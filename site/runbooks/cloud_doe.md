# Design of Experiments on GKE

The goal of this runbook is to explain how you can perform a
Design of Experiments (DoE) on Google Kubernetes Engine (GKE).

- [Assumptions and Limitations](#assumptions-and-limitations)
- [Getting started](#getting-started)
- [Examples](#examples)
- [Available nodepools](#available-nodepools)
- [Rule documentation](#rule-documentation)

## Assumptions and Limitations

- The bazel rule `doe_gke` presented in this runbook is ideal for executing an
  executable multiple times with different arguments. Each run of the executable
  is an independant *work item*.
- The executable must be provided inside a docker image available on the google
  cloud registry of project `testing`. It is also possible to use a bazel
  target for the image, in that case, the rule will handle the push on the registry.
- The executable is supposed to write a file named `DONE` in the output directory
  at the end of the job.
- The executable is supposed to wait for a file named `READY` in the input directory
  before starting the job (only if you are using `gcs_download`).

## Getting started

1.  Brezel is required as it contains the definition of the
    bazel rule [`doe_gke`](#rule-documentation).
2.  Build the `devel` image of brezel with all the required
    development tools: bazel, kubectl, etc.
    Note that if you are using a Research Project, the monorepo is already
    included as submodule and the project image already contains the required tools.
3.  Follow the getting started for using [Google Cloud Platform](gcloud.md).
4.  Make sure you have access to the GKE cluster `cluster`.
5.  Once gcloud and Kubernetes are properly configured, enter the development container using `make run-k8s`.

## Examples

### with parameter matrix

Let's demonstrate the `doe_gke` rule with project [UAM Fleet Scheduling][].

Assuming image `eu.gcr.io/testing/fleet-scheduling/runtime` is already
available on the registry, running a DoE with 4032 work items is as simple
as writing this `BUILD` file:

```
# fleet-scheduling/config/experiments/strategic_deconfliction/BUILD
load("@brezel//bazel/doe:gke.bzl", "doe_gke")

doe_gke(
    name = "experiment",
    image = "eu.gcr.io/testing/fleet-scheduling/runtime",
    command = "scala-fleetsched",
    gcs_upload = {"/out": "gs://data/experiments/scheduler/"},
    matrix = ":doe.mat",
    nodepool = "pool-compute-optimized-4",
)

genrule(
    name = "matrix",
    outs = ["doe.mat"],
    cmd = "for i in {0..4031}; do echo 4032 $$i $$((1500+i*1500/4031)) out >> $@ ; done"
)
```

Note that the parameters matrix `doe.mat` is generated by the bazel rule
named `matrix`, but it is also possible to provide it directly as a file
in the directory with the following content:

```text
4032 0 5 1500 out
[... skip 4029 lines ...]
4032 4030 5 2999 out
4032 4031 5 3000 out
```

Finally, dispatch the experiment on GKE with:

```bash
bazel run :experiment.create
```

Results can be downloaded from the bucket using [gsutil cp][].

[gsutil cp]: https://cloud.google.com/storage/docs/gsutil/commands/cp


### with jobs list

The second possibility to define DoE jobs is to use the `jobs` attribute instead
of the `matrix` attribute.

```
load("@io_bazel_rules_docker//python3:image.bzl", "py3_image")
load("@brezel//bazel/doe:gke.bzl", "doe_gke")

py3_image(
    name = "tailr-image",
    srcs = ["train_main.py"],
    layers = LAYERS,
    base = "@brezel//docker:python3_gpu_gke_base",
)

doe_gke(
    name = "tailr-doe",
    image = {"eu.gcr.io/testing/tailr/train:latest": ":tailr-image"},
    gcs_upload = {"/results": "gs://data/experiments/tailr_result/"},
    nodepool = "pool-gpu",
    jobs = {
        "tailr-loss": "--clfmode loss_init --clf_thresh 0.2",
        "tailr-fixed": "--clfmode fixed_init --classifier_init_period 3000",
        "tailr-cluster": "--clfmode cluster_count_init --classifier_init_period 2",
    },
    visibility = ["//visibility:public"],
)
```

The command line arguments of the work items are passed as values of the `jobs` dictionary
(it corresponds to the matrix rows in the previous example). The keys are used to name the
Kubernetes jobs. Note that if you want to set custom names to the jobs while using the `matrix`
file, you can use the `jobnames` attribute.


## Available nodepools

```eval_rst
+----------------------------+---------------------------------------+
| Node Pool                  | Machine type                          |
+============================+=======================================+
| pool-general               | `e2-standard-2`_                      |
+----------------------------+---------------------------------------+
| pool-experiments           | `n2-highmem-2`_                       |
+----------------------------+---------------------------------------+
| pool-small-experiments     | `n1-standard-1`_                      |
+----------------------------+---------------------------------------+
| pool-compute-optimized-[n] | `c2-standard-n`_                      |
+----------------------------+---------------------------------------+
| pool-gpu                   | `n1-standard-4`_ + `nvidia-tesla-t4`_ |
+----------------------------+---------------------------------------+
| pool-gpu-highmem           | `n1-highmem-4`_ + `nvidia-tesla-t4`_  |
+----------------------------+---------------------------------------+

.. _e2-standard-2: https://cloud.google.com/compute/docs/machine-types#e2_machine_types
.. _n1-standard-1: https://cloud.google.com/compute/docs/machine-types#n1_standard_machine_types
.. _n2-highmem-2: https://cloud.google.com/compute/docs/machine-types#n2_high-memory_machine_types
.. _c2-standard-n: https://cloud.google.com/compute/docs/machine-types#c2_machine_types
.. _n1-standard-4: https://cloud.google.com/compute/docs/machine-types#n1_standard_machine_types
.. _n1-hihgmem-4: https://cloud.google.com/compute/docs/machine-types#n1_high-memory_machine_types
.. _nvidia-tesla-t4: https://cloud.google.com/compute/docs/gpus#gpu_comparison_chart
```

## Rule documentation

<table>
  <tr>
    <th>Attributes</th>
    <th></th>
  </tr>

  <tr>
    <td><span class="pre highlight">name</span></td>
    <td>
      <a href="https://docs.bazel.build/versions/master/build-ref.html#name">Name</a>; required<br>
      A unique name for the target.<br>
      This attribute is used for naming the job, unless <span class="pre highlight">experiment</span> is given.
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">image</span></td>
    <td>
      String or Dict; required<br>
      The docker image of the worker container. Either pass the name of an image already available on the registry
      or if the image comes from a bazel rule, pass a string to label dictonoray to let the present rule upload the
      image on the registry. Syntax: <span class="pre">image = {"image_name": ":image_label"}</span>.
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">experiment</span></td>
    <td>
      String; optional<br>
      The name of the experiment. It appears in the kubernetes Job metadata.
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">command</span></td>
    <td>
      String; optional<br>
      The executable to run in the container.
      If not provided, Kubernetes will use the command included in the image.
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">gcs_upload</span></td>
    <td>
      Dict; required<br>
      The keys in this dictionary are the absolute paths of the folders
      containing the outputs files. The associated values are urls on GCS
      where the folders will be uploaded at the end of the job.<br>
      Only one item is currently supported.
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">gcs_download</span></td>
    <td>
      Dict; optional<br>
      The keys in this dictionary are urls on GCS that are going to be
      downloaded at the start of the job. The associated values are the
      folder absolute paths where the files should be downloaded.<br>
      Only one item is currently supported.
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">matrix</span></td>
    <td>
      <a href="https://docs.bazel.build/versions/master/build-ref.html#labels">Label</a>; required (unless <span class="pre highlight">jobs</span> is provided)<br>
      The file containing the parameters matrix of the DoE.
      One Job per row is created on Kubernetes. The columns are space-separated.
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">jobs</span></td>
    <td>
      <a href="https://docs.bazel.build/versions/master/build-ref.html#labels">Dict</a>; required (unless <span class="pre highlight">matrix</span> is provided)<br>
      The dictionary with the job names (dict keys) and their corresponding command line arguments (dict values).
      One item in the dictionary corresponds to one Job created on Kubernetes.
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">jobnames</span></td>
    <td>
      List; optional<br>
      String array with the jobnames (one name per matrix row). Cannot be combined with <span class="pre highlight">jobs</span>.
      If not given, the name of the job is equal to the experiment name with the job number as suffix.
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">deadline</span></td>
    <td>
      int; default: -1<br>
      Deadline in seconds to finish the job. Unlimited if -1.
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">nodepool</span></td>
    <td>
      String; Optional<br>
      The nodepool where the DoE should be run.<br>
      Allowed values: <span class="pre">"pool-general", "pool-experiments", "pool-small-experiments", "pool-compute-optimized-4", "pool-compute-optimized-8", "pool-compute-optimized-16", "pool-compute-optimized-60", "pool-gpu", "pool-gpu-highmem".
      <a href="#available-nodepools">More details here</a></span>
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">exclusive</span></td>
    <td>
      Boolean; default: False<br>
      If given, the job will take all the CPU resources available on the pod.<br>
      Only compatibale with <span class="pre highlight">nodepool = "pool-compute-optimized-{}"</span>
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">compress</span></td>
    <td>
      Boolean; default: True<br>
      Compress with gzip the files before uploading them on the bucket.
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">output</span></td>
    <td>
      String; <strong>deprecated, use <span class="pre highlight">gcs_upload</span></strong><br>
      The location of the outputs files. At the end of the job, the content
      of this folder will be uploaded to the bucket.<br>
    </td>
  </tr>

  <tr>
    <td><span class="pre highlight">bucket</span></td>
    <td>
      String; <strong>deprecated, use <span class="pre highlight">gcs_upload</span></strong><br>
      The GCS bucket where the results should be uploaded.<br>
    </td>
  </tr>

</table>

