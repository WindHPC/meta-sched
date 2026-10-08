# Meta Scheduler Documentation

## Table of Contents

- :world_map: [Overview](#overview)
- :construction: [Installation](#installation)
- :construction_worker: [Administrators](#administrators)
- :computer: [Users](#users)

## Overview

The core concepts of the Meta Scheduler are illustrated in the diagrams below:

<table><tbody style="text-align: center;">
<tr>
<td>Main components of the Meta Scheduler</td>
<td>Flowchar for job (array)</td>
<tr valign="top">
<td><img src="./diagrams/components.drawio.png" /></td>
<td><img src="./diagrams/job-flowchart.drawio.png" /></td>
</tr>
</tbody></table>

## Installation

1. Use [direnv](https://direnv.net/) or `source .envrc` to set up the project environment and to set up [uv](https://docs.astral.sh/uv/), if it isn't already installed.
2. During development, use `mscli-dev` and `msserver-dev` to test the CLI and server respectively.
3. Run static type checking, linting and formatting, and build the installable Python package under `./src/<component>/dist/` using `package <all|client|server>`
4. Install the Python packages using `install <all|client|server>`. This also installs Python 3.12 (through [pyenv](https://github.com/pyenv/pyenv)) and [pipx](https://pipx.pypa.io/stable/).

## Administrators

After [installing the server package](#installation), execute it using `MS_API_KEY=someSecret msserver`. 
A [default configuration file](../src/server/server/config/default.toml) is displayed, if no configuration is found at `/etc/meta-sched.toml`.
(Alternatively, a different path can be used with `--config <path/to/config.toml>`.)
If the application is running under a non-root user and requires `sudo`, add the flag `--sudo`.

To persistently execute the server in the background after the system has booted, create a new [systemd unit](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) or use a terminal multiplexer like [tmux](https://github.com/tmux/tmux/wiki) to manually run it in the background.

The state of the Meta Scheduler can be updated with the state of the targets using

```shell
(
  export MS_API_KEY=someSecret
  # Optional:
  export DATACENTER_API_USER=someUser
  export DATACENTER_API_PASS=somePassword

  msprobe [-t <target ID>,...] [-i <interval>]
)
```

from the client package.
This state can be used by the scheduling policy to optimize job distribution across targets.

The structure of the configuration file is explained by example below:

```toml
host = "localhost"
port = 8001
# Use ephemeral in-memory database or flat file (e.g. "sqlite:////var/opt/meta-sched.db")
db_url = "sqlite://" 

# Time between invoking the scheduling policy
scheduling_loop_interval = 10
# Scheduling policy
scheduler_class_name = "stochastic.py:WeightedByCoresAvailability"
# Override default parameters of the scheduling policy
[scheduler_parameter_overrides]
epsilon = 1e-9
threshold_reliability_renewable = 0.8

# List of targets available for scheduling
[[targets]]
id             = "hsuper-small"
host           = "hsuper-login01.hsu-hh.de"
batch_system   = "slurm" # Alternative: pbs, none
queue          = "small" # (The Slurm partition)
constraints    = "cpu8360Y&ram256" # Constraints expression used by Slurm (ignored by others)
nodes          = 571
cores_per_node = 72
max_time       = "72:00:00" # Formated as d-hh:MM:ss
max_nodes      = 5
min_nodes      = 1
tags           = ["x86"]
# Specify concrete environment modules available
[targets.module_map]
MPI            = "intel-oneapi-mpi"
# MPI            = "gcc/14.2.0 openmpi/4.1.7" # Hierarchical module system (If e.g. OpenMPI depends on GCC)
# Optionally source system wide scripts
# source_scripts = ["/etc/profile"]

# Additional targets below
[[targets]]
id             = "windhpc-hlrs"
# ...
```

Scheduling decisions are made independently for each job of an array in two steps:

1. **Filter** suitable target subset (client side).
2. **Select** execution target (server side).  
This is done by the scheduling policy which may be either stateful or stateless (stochastic).  

The scheduling policy implements [`Policy`](/src/ms_server/scheduling/__init__.py) and is specified using the Python module file (relative to [`/src/ms_server/scheduling/`](/src/ms_server/scheduling/) or as an absolute path) and class name in the configuration:

```toml
scheduler_class_name = "stochastic.py:WeightedByCores"
```

### Required Software on the Targets

- Posix shell (e.g. Bash)
- rsync or SFTP server
- SSH server
- tail, touch, mkdir, rm, date, sed (coreutils)
- xargs (findutils)
- grep
- awk
- qsub, qdel, qstat, pbsnodes (For PBS Pro/OpenPBS)
- sbatch, scancel, squeue, sacct (For Slurm)
- flock (For targets without a batch system)
- module load (Optional, Environment Modules)

## Users

After [installing the client package](#installation), submit job arrays using `mscli`.

### Folder Structure

#### Submit Host

On the submit host, all data lives in `$HOME/meta-sched/`.
For each `job spec` this directory contains a `<job spec>/input/` folder with job input files which are uploaded to the target hosts and `<job spec>/output/` folder containing the subdirectories `<array_id>_<array_idx>/` for the output of the corresponding jobs.  
:warning: Do **NOT** delete any files from the output directory before the corresponding job has been completed, in particular `.pid` (Handle to job process on the submit host), `.status` status of the job, `output` and `error` (streaming stdout/stderr output).

#### Target Host

On the target host, all data lives in `$HOME/.meta-sched/`. It mirrors the contents of the main folder on the submit host:  
For each `job spec` this directory contains a `<job spec>/input/` folder where job input files are uploaded to and `<job spec>/output/` folder containing the subdirectories `<array_id>_<array_idx>/` which are the working and output directories of the corresponding jobs.

### Configuration

Upon running the command for the first time, the default configuration file at `$HOME/.config/meta-sched.toml` is created.  
This file contains the endpoint of the Meta Scheduler server component used by the client.
For additional filtering of targets used for scheduling of a particular job, custom tags or source scripts can be added to the user configuration:

```toml
[[targets]]
id   = "windhpc-hlrs"
tags = ["test"]
source_scripts = ["~/spack/share/spack/setup-env.sh"]

# Fields used by msprobe to gather additional data about the target
datacenter_api_endpoint = "https://example.org/api/v1"
datacenter_api_forecast_source_id = 0
```

First, run `mscli ssh-config` to create/update `~/.ssh/config.d/meta-sched` with the targets of the Meta Scheduler server.  
Then, edit this SSH configuration file (e.g. using `vim` or `nano`), providing credentials to the target systems which should be used to execute jobs. (Test this by manually connecting to target using the SSH configuration entry name.)

### Job Control

1. To create a new job spec, run `mscli create <template> <job spec>` where `template` may be `hello-<array|AutoPas|ls1|mamico|mpi|ms2>`. (Check the [corresponding source code folder](../src/ms_client/data/examples/jobs/).)

2. Modify the job files template instantiated at `$HOME/meta-sched/jobs/<job spec>`.

3. Submit the job array by executing `mscli submit <job spec>`

4. Use `mscli status [--all]` to display the status of submitted jobs.

5. To cancel a job that has not yet completed, use `mscli cancel <job id>`.

### Job Specifications

The job spec in `$HOME/meta-sched/jobs/<job spec>/spec.toml` defines the requirements of a job array, the number of jobs, the input files, and the command to be executed.  
An example is given below:

```toml
# Fetch source code in case target has no internet access.
# The current working directory is $HOME/meta-sched/jobs/<job spec>.
# Note: The local setup step is executed just after the job is submitted for scheduling.
cmd_setup_local  = "git clone --depth 1 https://example.org/repo.git $MS_INPUT"
# Compile source code copied from $HOME/meta-sched/jobs/<job spec>/input/
# The current working directory is the job output folder ($MS_OUTPUT).
# (Here the binary will be included in the results downloaded from the target unless deleted.)
# Note: The target setup step is only executed after the job has been scheduled on a target.
cmd_setup_target = "mpicxx $MS_INPUT/example.cpp -o example"
# Run the example three times on two nodes through the batch system.
# (Each job in the array may run on a different target.)
cmd_main         = "mpiexec ./example --seed $MS_ARRAY_IDX $MS_VARS_scenario"
array_size       = 3
nodes            = 2
ranks_per_node   = 1
# Optionally, supply number of cores (OpenMP) or request the full node (exclusive).
# cores_per_rank   = 12
# exclusive        = False
# Only use targets providing an MPI implementation module like OpenMPI or MPICH. (Will be loaded.)
required_modules = ["MPI"]
# Provide additional variables to the job environment (MS_VARS_<key>) or the time expression below. (Numeric only.)
vars = { scenario = "small" }
# Provide required wall time in the format d-hh:MM:ss or as 'seconds = <seconds>'.
# To determine the number of seconds dynamically with SymPy, use 'time = "= <expression>" where p is the total number of cores available and i is the job index in the array.
time             = "0-00:05:00" 
# Only consider a subset of targets (e.g. using renewable energy).
required_tags    = ["green"]
```

The job spec can be validated with `mscli validate <job spec>`.

### Toubleshooting

If you encounter `No targets available to run job spec: <job spec>`, turn on debugging to investigate:

```shell
$ MS_DEBUG_FILTER_TARGETS=1 mscli-dev submit <job spec>
[DEBUG]: Can the job run on hsuper-small? No. (Required tag "green" missing.)
[DEBUG]: Can the job run on windhpc-wwit? No. (Too many cores required.)
# ...
```

Use `export MS_DEBUG_CMD=1` to print the commands that are run locally and remotely to the `sterr` file of the job output.

If a command is not found, consider adding the following to the target in the server configuration:

```toml
# SSH connections use non-interactive shell
source_scripts = ["/etc/profile"]
```
