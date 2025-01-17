# DvC support

These tests make use of the `dvc` example.

This project is used as a template for a working DvC repo. We run
`setup.py` to generate a project we can work with.

    >>> tmp = mkdtemp()
    >>> cd(example("dvc"))
    >>> run("python setup.py '%s'" % tmp)
    Initializing ...
    Initializing Git
    Initializing DvC
    Copying source code files
    <exit 0>

    >>> cd(tmp)

Operations supported by the project:

    >>> run("guild ops")  # doctest: +REPORT_UDIFF
    eval-models-dvc-dep        Use Guild to run the eval models operation
    eval-models-dvc-stage      Use Guild DvC plugin to run eval-models stage
    faketrain-dvc-stage        Use Guild DvC plugin to run faketrain stage
    hello-dvc-dep              Uses DvC dependency to fetch required file if needed
    hello-dvc-dep-always-pull  Uses DvC dependency to always fetch required file
    hello-dvc-stage            Uses Guild DvC plugin to run hello stage
    hello-guild-op             Standard Guild dependency example without DvC support
    prepare-data-dvc-dep       Use DvC dependency to fetch required file if needed
    prepare-data-dvc-stage     Use Guild DvC plugin to run prepare-data stage
    train-models-dvc-dep       Use Guild to run the train models operation
    train-models-dvc-stage     Use Guild DvC plugin to run train-models stage
    <exit 0>

## DvC resource sources

The dvc plugin adds support for a 'dvc' source type. This is used in
the sample project's 'hello-dvc-dep' operation.

The 'dvc' source type acts like a 'file' source when the specified
file is available.

    >>> write("hello.in", "Project File")

    >>> run("guild run hello-dvc-dep -y")
    Resolving dvcfile:hello.in dependency
    Hello Project File!
    <exit 0>

If the file isn't available, Guild uses 'dvc pull' to fetch it.

    >>> rm("hello.in")

    >>> run("guild run hello-dvc-dep -y")
    Resolving dvcfile:hello.in dependency
    Fetching DvC resource hello.in
    A       hello.in
    1 file added and 1 file fetched
    Hello World!
    <exit 0>

In this case, Guild copies 'hello.in.dvc' to the run directory from
the project directory. This is in turn used by 'dvc pull' to fetch
'hello.in' from remote storage.

    >>> run("guild ls -n")
    hello.in
    hello.in.dvc
    hello.out
    <exit 0>

Remote storage is defined in the project file 'dvc.config.in'. This is
a special file that Guild uses to initialize the DvC repo.

    >>> cat(join_path(tmp, "dvc.config.in"))
    [core]
        analytics = false
        remote = guild-pub
    ['remote "guild-pub"']
        url = https://guild-pub.s3.amazonaws.com/dvc-store
    ['remote "guild-s3"']
        url = s3://guild-pub/dvc-store

    >>> run("guild cat -p .dvc/config")
    [core]
        analytics = false
        remote = guild-pub
    ['remote "guild-pub"']
        url = https://guild-pub.s3.amazonaws.com/dvc-store
    ['remote "guild-s3"']
        url = s3://guild-pub/dvc-store
    <exit 0>

A DvC dependency can always be pulled by setting 'always-pull' to true
in the Guild file. The operation 'hello-dvc-dep-always-pull'
illustrates this.

Provide 'hello.in' in the project directory.

    >>> write("hello.in", "Ignored Local File")
    >>> run("guild run hello-dvc-dep-always-pull -y")
    Resolving dvcfile:hello.in dependency
    Fetching DvC resource hello.in
    A       hello.in
    1 file added and 1 file fetched
    Hello World!
    <exit 0>

## Use of `guild.plugins.dvc_stage_main`

The module `guild.plugins.dvc_stage_main` can be used to run a stage
defined in dvc.yaml.

These operations use this module as their `main` attr:

- prepare-data-dvc-stage
- train-models-dvc-stage
- eval-models-dvc-stage

Guild handles dependency resolution from this module as follows:

- Dependencies that are resolved from the project are copied from the
  project dir (if present) or pulled using the DvC config file for the
  dependency.

- Dependencies that are generated by upstream stages are resolved by
  looking for Guild operations for those stages. Such operation are
  designated by a `dvc-stage` run attribute matching the upstream
  stage name.

To illustrate, we first delete existing runs.

    >>> quiet("guild runs rm -y")

Try to run operations that requires an upstream stage.

    >>> run("guild run train-models-dvc-stage -y")
    INFO: [guild] Initializing run
    INFO: [guild] Copying train_models.py
    guild: no suitable run for stage 'prepare-data' (needed for iris.npy)
    <exit 1>

    >>> run("guild run eval-models-dvc-stage -y")
    INFO: [guild] Initializing run
    INFO: [guild] Copying eval_models.py
    guild: no suitable run for stage 'prepare-data' (needed for iris.npy)
    <exit 1>

Run op required for 'prepare-data'.

    >>> run("guild run prepare-data-dvc-stage -y")  # doctest: +REPORT_UDIFF
    INFO: [guild] Initializing run
    INFO: [guild] Fetching iris.csv
    INFO: [guild] Fetching DvC resource iris.csv
    A       iris.csv
    1 file added and 1 file fetched
    INFO: [guild] Copying prepare_data.py
    INFO: [guild] Running stage 'prepare-data'
    Saving iris.npy
    <exit 0>

This resolved the required file 'iris.csv' by pulling from the remote
storage.

    >>> run("guild ls -n")  # doctest: +REPORT_UDIFF
    dvc.lock
    dvc.yaml
    iris.csv
    iris.csv.dvc
    iris.npy
    prepare_data.py
    <exit 0>

With the 'prepare-data' operation, we can run 'train-models'.

    >>> run("guild run train-models-dvc-stage -y")  # doctest: +REPORT_UDIFF
    INFO: [guild] Initializing run
    INFO: [guild] Copying train_models.py
    INFO: [guild] Using ... for 'prepare-data' DvC stage dependency
    INFO: [guild] Linking iris.npy
    INFO: [guild] Copying params.json.in
    INFO: [guild] Running stage 'train-models'
    C=1.000000
    gamma=0.700000
    max_iters=10000.000000
    Saving model-1.joblib
    Saving model-2.joblib
    Saving model-3.joblib
    Saving model-4.joblib
    <exit 0>

    >>> run("guild ls -n")  # doctest: +REPORT_UDIFF
    dvc.lock
    dvc.yaml
    iris.npy
    model-1.joblib
    model-2.joblib
    model-3.joblib
    model-4.joblib
    params.json.in
    train_models.py
    <exit 0>

And with 'train-models' and 'prepare-data', we can run 'eval-models'
successfully.

    >>> run("guild run eval-models-dvc-stage -y")  # doctest: +REPORT_UDIFF
    INFO: [guild] Initializing run
    INFO: [guild] Copying eval_models.py
    INFO: [guild] Using ... for 'prepare-data' DvC stage dependency
    INFO: [guild] Linking iris.npy
    INFO: [guild] Using ... for 'train-models' DvC stage dependency
    INFO: [guild] Linking model-1.joblib
    INFO: [guild] Linking model-2.joblib
    INFO: [guild] Linking model-3.joblib
    INFO: [guild] Linking model-4.joblib
    INFO: [guild] Copying params.json.in
    INFO: [guild] Running stage 'eval-models'
    plot_spacing=0.400000
    Saving models-eval.json
    Saving models-eval.png
    INFO: [guild] Logging metrics from models-eval.json
    <exit 0>

    >>> run("guild ls -n")  # doctest: +REPORT_UDIFF
    dvc.lock
    dvc.yaml
    eval_models.py
    events.out.tfevents...
    iris.npy
    model-1.joblib
    model-2.joblib
    model-3.joblib
    model-4.joblib
    models-eval.json
    models-eval.png
    params.json.in
    <exit 0>

### Logging summaries from metrics

After a DvC stage is run, Guild reads any metrics generated by the
stage and logs the values as TF summaries.

The eval operation writes metrics to 'models-eval.json'. This is
defined in 'dvc.yaml'.

    >>> run("guild cat -p dvc.yaml")
    stages:
      ...
      eval-models:
        ...
        metrics:
        - models-eval.json:
            cache: false
        ...
    <exit 0>

    >>> run("guild ls -np models-eval.json")
    models-eval.json
    <exit 0>

Guild reads the values and writes them summaries.

    >>> run("guild runs info")
    id: ...
    operation: eval-models-dvc-stage
    ...
    scalars:
      modle-1-score: 0... (step 0)
      modle-2-score: 0... (step 0)
      modle-3-score: 0... (step 0)
      modle-4-score: 0... (step 0)
    <exit 0>

## Guild simulated stage with param flags

The operation 'train-models-dvc-dep' uses a standard Guild operation
without DvC to run the training operation. This uses 'params.json.in'
for the args dest. This is consistent with the DvC use of params.

    >>> run("guild run train-models-dvc-dep train.C=2.0 -y")
    Resolving config:params.json.in dependency
    Resolving dvcstage:prepare-data dependency
    Using run ... for dvcstage:prepare-data resource
    C=2.000000
    gamma=0.700000
    max_iters=10000.000000
    Saving model-1.joblib
    Saving model-2.joblib
    Saving model-3.joblib
    Saving model-4.joblib
    <exit 0>

## Batches with a DvC stage

The 'faketrain' DvC stage can be run using the 'faketrain-dvc-stage'
operation. We can use the flag support to run a batch.

    >>> run("guild run faketrain-dvc-stage x=[-1.0,0.0,1.0] -y")  # doctest: +REPORT_UDIFF
    INFO: [guild] Running trial ...: faketrain-dvc-stage (noise=0.1, x=-1.0)
    INFO: [guild] Resolving config:params.json.in dependency
    INFO: [guild] Initializing run
    INFO: [guild] Copying faketrain.py
    INFO: [guild] Running stage 'faketrain'
    x: -1.000000
    noise: 0.100000
    loss: ...
    INFO: [guild] Running trial ...: faketrain-dvc-stage (noise=0.1, x=0.0)
    INFO: [guild] Resolving config:params.json.in dependency
    INFO: [guild] Initializing run
    INFO: [guild] Copying faketrain.py
    INFO: [guild] Running stage 'faketrain'
    x: 0.000000
    noise: 0.100000
    loss: ...
    INFO: [guild] Running trial ...: faketrain-dvc-stage (noise=0.1, x=1.0)
    INFO: [guild] Resolving config:params.json.in dependency
    INFO: [guild] Initializing run
    INFO: [guild] Copying faketrain.py
    INFO: [guild] Running stage 'faketrain'
    x: 1.000000
    noise: 0.100000
    loss: ...
    <exit 0>

    >>> run("guild compare -t -cc .operation,.status,.label,=noise,=x,loss -n3")
    run  operation            status     label             noise  x     loss
    ...  faketrain-dvc-stage  completed  noise=0.1 x=1.0   0.1    1.0   ...
    ...  faketrain-dvc-stage  completed  noise=0.1 x=0.0   0.1    0.0   ...
    ...  faketrain-dvc-stage  completed  noise=0.1 x=-1.0  0.1    -1.0  ...
    <exit 0>

## Cross op-style dependencies

There are two operation styles in the 'dvc' sample project:

- Standard Guild that provide the same functionality as DvC stages
- Wrapped DvC stages

It's possible for a Guild operation (non-wrapped DvC op) to provide a
DvC stage by writing a 'dvc:stage' run attribute with the applicable
stage name.

Here's an example. We first delete the current runs.

    >>> quiet("guild runs rm -y")

When we run a downstream train DvC stage, it fails because we don't
have prepared data.

    >>> run("guild run train-models-dvc-stage -y")
    INFO: [guild] Initializing run
    INFO: [guild] Copying train_models.py
    guild: no suitable run for stage 'prepare-data' (needed for iris.npy)
    <exit 1>

We can satisfy this requirement using a non-DvC operation that
provides the required prepared data files (the operation uses a 'dvc'
resource type but does not run as a DvC stage).

    >>> run("guild run prepare-data-dvc-dep -y")
    Resolving dvcfile:iris.csv dependency
    Fetching DvC resource iris.csv
    A       iris.csv
    1 file added and 1 file fetched
    Saving iris.npy
    <exit 0>

This operation writes a 'dvc:stage' attribute that indicates that the
run provides the output files for the 'prepare-data' stage.

    >>> run("guild runs info")
    id: ...
    operation: prepare-data-dvc-dep
    ...
    dvc-stage: prepare-data
    ...
    <exit 0>

The DvC train stage can use this run to satisfy its dependency.

    >>> run("guild run train-models-dvc-stage -y")
    INFO: [guild] Initializing run
    INFO: [guild] Copying train_models.py
    INFO: [guild] Using ... for 'prepare-data' DvC stage dependency
    INFO: [guild] Linking iris.npy
    INFO: [guild] Copying params.json.in
    INFO: [guild] Running stage 'train-models'
    C=1.000000
    gamma=0.700000
    max_iters=10000.000000
    Saving model-1.joblib
    Saving model-2.joblib
    Saving model-3.joblib
    Saving model-4.joblib
    <exit 0>

## Run DvC stage directly

Guild supports running a stage defined in dvc.yaml directly using the
operation name syntax 'dvc.yaml:<stage>'.

Here's the op help for the 'faketrain' stage.

    >>> run("guild run dvc.yaml:faketrain --help-op")
    Usage: guild run [OPTIONS] dvc.yaml:faketrain [FLAG]...
    <BLANKLINE>
    Stage 'faketrain' imported from dvc.yaml
    <BLANKLINE>
    Use 'guild run --help' for a list of options.
    <BLANKLINE>
    Flags:
      x  (default is 0.1)
    <exit 0>

Note that Guild imports the required stage params as flags.

We can run the stage as an operation, including as a batch.

    >>> run("guild run dvc.yaml:faketrain x=[0.2,0.3] -y")  # doctest: +REPORT_UDIFF
    INFO: [guild] Running trial ...: dvc.yaml:faketrain (x=0.2)
    INFO: [guild] Resolving dvcfile:faketrain.py dependency
    INFO: [guild] Resolving config:params.json.in dependency
    INFO: [guild] Initializing run
    INFO: [guild] Running stage 'faketrain'
    x: 0.200000
    noise: 0.100000
    loss: ...
    INFO: [guild] Running trial ...: dvc.yaml:faketrain (x=0.3)
    INFO: [guild] Resolving dvcfile:faketrain.py dependency
    INFO: [guild] Resolving config:params.json.in dependency
    INFO: [guild] Initializing run
    INFO: [guild] Running stage 'faketrain'
    x: 0.300000
    noise: 0.100000
    loss: ...
    <exit 0>

When run directly, the run has a 'dvc-stage' attribute, which
specifies the stage.

    >>> run("guild runs info")
    id: ...
    operation: dvc.yaml:faketrain
    ...
    dvc-stage: faketrain
    tags:
    flags:
      x: 0.3
    scalars:
      loss: ... (step 0)
      noise: ... (step 0)
    <exit 0>

### Dependencies

When run directly, Guild sets up dependencies between stage
operations. These are used to resolve required files from upstream
operations/stages.

Delete the runs for the following tests.

    >>> quiet("guild runs rm -py")

Let's try to run the train stage, which depends on prepare data.

    >>> run("guild run dvc.yaml:train-models -y")
    WARNING: cannot find a suitable run for required resource 'dvcstage:prepare-data'
    Resolving dvcfile:train_models.py dependency
    Resolving dvcstage:prepare-data dependency
    guild: run failed because a dependency was not met: could not resolve
    'dvcstage:prepare-data' in dvcstage:prepare-data resource: no suitable
    run for 'prepare-data' stage
    <exit 1>

Now run the required prepare data stage.

    >>> run("guild run dvc.yaml:prepare-data -y")
    Resolving dvcfile:iris.csv dependency
    Fetching DvC resource iris.csv
    A       iris.csv
    1 file added and 1 file fetched
    Resolving dvcfile:prepare_data.py dependency
    INFO: [guild] Initializing run
    INFO: [guild] Running stage 'prepare-data'
    Saving iris.npy
    <exit 0>

And run the train stage again.

    >>> run("guild run dvc.yaml:train-models -y")
    Resolving dvcfile:train_models.py dependency
    Resolving dvcstage:prepare-data dependency
    Using run ... for dvcstage:prepare-data resource
    Resolving config:params.json.in dependency
    INFO: [guild] Initializing run
    INFO: [guild] Running stage 'train-models'
    C=1.000000
    gamma=0.700000
    max_iters=10000.000000
    Saving model-1.joblib
    Saving model-2.joblib
    Saving model-3.joblib
    Saving model-4.joblib
    <exit 0>
