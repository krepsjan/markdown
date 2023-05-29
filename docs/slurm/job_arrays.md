# Job Arrays

## Introduction

Job array is a collection of jobs that run the same batch script.
The jobs in the array are differentiated by the values
of the environment variable `SLURM_ARRAY_TASK_ID`.

From the Slurm documentation page
[Job Array Support](https://slurm.schedmd.com/job_array.html):

> Job arrays offer a mechanism for submitting and managing collections of similar jobs quickly and easily

## Basic usage

To submit a job array, use the option `sbatch --array`, for example:

```bash
sbatch --array=0-999 run.sh
```

This call creates 1000 jobs (or tasks) that belong to one job array.
Each of the 1000 tasks will run the script `run.sh`
with the environment variable `SLURM_ARRAY_TASK_ID`
set to a unique number in the range from 0 to 999 (inclusive).
The script `run.sh` is expected to differentiate its functionality
based on the value of `SLURM_ARRAY_TASK_ID`.

Some additional environment variables are available
in batch scripts run in job arrays, for example:

- `SLURM_ARRAY_JOB_ID`: First job ID of the array. Identifies the array.
- `SLURM_ARRAY_TASK_MAX`: Highest task index in this array.
  In the example above, this would evaluate to 999.

The maximum number of tasks in an array is limited by two
[Slurm configuration parameters](https://slurm.schedmd.com/slurm.conf.html):

- `MaxArraySize` (value as of 2020-02-18: 17000) <!--`scontrol show config | grep MaxArraySize`-->
- `MaxJobCount` (value as of 2020-02-18: 170000) <!--`scontrol show config | grep MaxJobCount`-->

## Task parameterization

In a common use case,
we need to execute the command `process` with many different sets of parameters.
To use custom sets of parameters,
prepare the file `parameters.txt` with one set of parameter values per line
and run:

```bash
PARAMETERS=${PARAMETERS:-parameters.txt}
PARAMETERS_COUNT=$(wc -l < "$PARAMETERS")
# Extract maximum array size from Slurm configuration.
MAX_ARRAY_SIZE=$(scontrol show config | grep MaxArraySize | awk '{split($0, a, "="); print a[2]}' | sed 's/^ *//g')
ARRAY_TASK_COUNT=$((MAX_ARRAY_SIZE < PARAMETERS_COUNT ? MAX_ARRAY_SIZE : PARAMETERS_COUNT))
ARRAY=0-$((ARRAY_TASK_COUNT - 1))

sbatch --input="$PARAMETERS" --array="$ARRAY" run.sh
```

`run.sh`:

```bash
# If there are more input lines than tasks in the array, we need to process
# multiple input lines in one task. We do that by processing all the lines with
# 1-based index congruent to
# $((SLURM_ARRAY_TASK_ID + 1)) modulo $((SLURM_ARRAY_TASK_MAX + 1)).

sed -n "$((SLURM_ARRAY_TASK_ID + 1))~$((SLURM_ARRAY_TASK_MAX + 1))p" | xargs process
```

!!! Note
    The scripts may be simplified in case we assume that
    the number of sets of parameters will never surpass the maximum array size.

## Analyzing results with `sacct`

Using large collections of jobs clutters the output of
[`sacct`](https://slurm.schedmd.com/sacct.html).

### Displaying only one task per array

Use the following command as a drop-in replacement of `sacct`
that only prints the task with the index 0 from each job array:

```bash
sacct_0()
{
  sacct_output=$(sacct "$@")
  echo "$sacct_output" | head -n2
  echo "$sacct_output" | grep --color=never '^[[:digit:]]\{6\}\(_0\)\?\>'
}
```

For example, to display a concise list of jobs executed from 2020-01-01, run:

```bash
sacct_0 -X -S 2020-01-01
```

### Aggregating statistics of job collection

The following command accepts the same parameters as `sacct`
and aggregates the data from the corresponding jobs.
Notably ti displays:

- Distribution of job states (`State`)
- Maximum elapsed time (`Elapsed`)
- Maximum resident set size (`MaxRSS`)

!!! Warning
    The command depends on the non-standard utility
    [xsv](https://github.com/BurntSushi/xsv).

```bash
sacct_stats()
{
  sacct -P --delimiter ',' -X -o State,ExitCode,Timelimit,Partition,NodeList "$@" | xsv frequency | xsv table
  sacct -P --delimiter ',' -o JobID,ReqMem --noconvert "$@" | xsv search -s JobID ".*\.batch" | xsv select ReqMem | xsv frequency | xsv table
  sacct -P --delimiter ',' -X -o JobID,Start,End,Elapsed --noconvert "$@" | xsv select Start,End,Elapsed | xsv stats | xsv select field,min,max | xsv table
  sacct -P --delimiter ',' -o JobID,MaxRSS --noconvert "$@" | xsv search -s JobID ".*\.batch" | xsv select MaxRSS | xsv stats | xsv select field,min,max,mean,stddev | xsv table
}
```

`sacct_stats` aggregates the following `sacct` fields:

- Categorical (aggregation: frequency table):
    - State
    - ExitCode
    - Timelimit
    - Partition
    - NodeList
    - ReqMem
- Numeric (aggregation: min, max):
    - Start
    - End
    - Elapsed
    - MaxRSS (additional aggregation: mean, stddev)

For example, running `sacct_stats -j 451220` prints the statistics
of the array job 451220.
Example output:

```
$ sacct_stats -j 451220
field      value      count
State      COMPLETED  75
State      FAILED     24
State      TIMEOUT    1
ExitCode   0:0        76
ExitCode   123:0      24
Timelimit  05:34:00   100
Partition  compute    100
NodeList   node-08    29
NodeList   node-03    28
NodeList   node-11    15
NodeList   node-04    14
NodeList   node-05    14
field   value   count
ReqMem  8320Mc  100
field    min                  max
Start    2020-02-11T18:09:19  2020-02-11T18:17:49
End      2020-02-11T18:11:50  2020-02-11T23:51:44
Elapsed  00:00:54             05:34:06
field   min       max         mean       stddev
MaxRSS  99315712  6388080640  807832576  1350328795.654066
```
