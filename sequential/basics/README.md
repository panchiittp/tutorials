[![By ULHPC](https://img.shields.io/badge/by-ULHPC-blue.svg)](https://hpc.uni.lu) [![Licence](https://img.shields.io/badge/license-GPL--3.0-blue.svg)](http://www.gnu.org/licenses/gpl-3.0.html) [![GitHub issues](https://img.shields.io/github/issues/ULHPC/tutorials.svg)](https://github.com/ULHPC/tutorials/issues/) [![](https://img.shields.io/badge/slides-PDF-red.svg)](https://github.com/ULHPC/tutorials/raw/devel/basic/sequential_jobs/slides.pdf) [![Github](https://img.shields.io/badge/sources-github-green.svg)](https://github.com/ULHPC/tutorials/tree/devel/basic/sequential_jobs/) [![Documentation Status](http://readthedocs.org/projects/ulhpc-tutorials/badge/?version=latest)](http://ulhpc-tutorials.readthedocs.io/en/latest/basic/sequential_jobs/) [![GitHub forks](https://img.shields.io/github/stars/ULHPC/tutorials.svg?style=social&label=Star)](https://github.com/ULHPC/tutorials)

# HPC Management of Sequential and Embarrassingly Parallel Jobs

     Copyright (c) 2013-2019 UL HPC Team <hpc-sysadmins@uni.lu>

[![](https://github.com/ULHPC/tutorials/raw/devel/basic/sequential_jobs/cover_slides.png)](https://github.com/ULHPC/tutorials/raw/devel/basic/sequential_jobs/slides.pdf)


For many users, the reason to consider (or being encouraged) to offload their computing executions on a (remote) HPC or Cloud facility is tied to the limits reached by their computing devices (laptop or workstation).
It is generally motivated by time constraints:

> "_My computations take several hours/days to complete. On an HPC, it will last a few minutes, no?_"

or search-space extensions:

> "_I need to check my application against a **huge** number of input pieces (files) - it worked on a few of them locally but takes ages for a single check. How to proceed on HPC?_"

For several of you, the application required for your research that you traditionally run on your laptop or workstation consists in:

* a (well-known) application installed on your system, iterated over multiple input conditions configured by specific command-line arguments / configuration files
* a compiled program (C, C++, Java, Go etc.), iterated over multiple input conditions
* your favorite R or python (custom) development scripts, iterated again over multiple input conditions

**`/!\ IMPORTANT:` **Be aware that in most of the cases, these application are inherently SERIAL**: These are **able to use only one core** when executed.


--------------------
## Pre-requisites ##

Ensure you are able to [connect to the UL HPC clusters](https://hpc.uni.lu/users/docs/access.html).
In particular, recall that the `module` command **is not** available on the access frontends.

If you have never configured [GNU Screen](http://www.gnu.org/software/screen/) before, and while not strictly mandatory, we advise you to rely on our customized configuration file for screen [`.screenrc`](https://github.com/ULHPC/dotfiles/blob/master/screen/.screenrc) available on [Github](https://github.com/ULHPC/dotfiles/blob/master/screen/.screenrc) and available on the access nodes under `/etc/dotfiles.d/screen/.screenrc`

```bash
### Access to ULHPC cluster - here iris
(laptop)$> ssh iris-cluster
# /!\ Advanced (but recommended) best-practice:
#    always work within an GNU Screen session named with 'screen -S <topic>' (Adapt accordingly)
# IIF not yet done, copy ULHPC .screenrc in your home
(access-iris)$> cp /etc/dotfiles.d/screen/.screenrc ~/
```

Now you'll need to pull the latest changes in your working copy of the [ULHPC/tutorials](https://github.com/ULHPC/tutorials) you should have cloned in `~/git/github.com/ULHPC/tutorials` (see ["preliminaries" tutorial](../../preliminaries/))

``` bash
(access-iris)$> cd ~/git/github.com/ULHPC/tutorials
(access-iris)$> git pull
```

Now **configure a dedicated directory `tutorials/sequential` for this session**

``` bash
# return to your home
(access-iris)$> mkdir -p tutorials/sequential
(access-iris)$> cd tutorials/sequential
# create a symbolic link to the reference material
(access-iris)$> ln -s ~/git/github.com/ULHPC/tutorials/sequential ref.d
```

**Advanced users** (_eventually_ yet __strongly__ recommended), create a [GNU Screen](http://www.gnu.org/software/screen/) session you can recover later - see ["Getting Started" tutorial](../../getting-started/)

``` bash
# /!\ Advanced (but recommended) best-practice:
#     Always work within an GNU Screen session named with 'screen -S <topic>' (Adapt accordingly)
#     _note_: assumes you have previously copied ULHPC .screenrc in your homedir
(access-iris)$> screen -S HPC-school
#    CTRL + a c: (create) creates a new Screen window. The default Screen number is zero.
#    CTRL + a n: (next) switches to the next window.
#    CTRL + a p: (prev) switches to the previous window.
#    CTRL + a A: (title) rename the current window
#    CTRL + a d: (detach) detaches from a Screen -
# Once detached:
#   screen -ls : list available screen
#   screen -x    reattach to a past screen
```

## A sample "Stress Me!" exploration

In this sample scenario, we will first mimic serial usage for tasks lasting different amount time (here, ranging from 1 to 30s). This is very typical: your serial application is likely to take more or less time depending on its execution context (typically controlled by command line arguments).

You will use a script, `scripts/stressme`, to reproduce a configurable amount of  stress on the system set for a certain amount of time (by default: 20s) passed as parameter using the '[stress](https://linux.die.net/man/1/stress)' command.
We would like thus to conduct the following executions:

``` bash
stressme 1     # execution time: 1s
stressme 2     # execution time: 2s
stressme 3     # execution time: 3s
stressme 4     # execution time: 4s
stressme 5     # execution time: 5s
stressme 6     # execution time: 6s
stressme 7     # execution time: 7s
[...]
stressme 27    # execution time: 27s
stressme 28    # execution time: 28s
stressme 29    # execution time: 29s
stressme 30    # execution time: 30s
```

Running this job campaign **sequentially** (one after the other on the same core) would take approximatively **465s**.
The below table wrap up the sequential times required for the completion of the job campaign depending on the number of `stressme` tasks, and the _theoretical_ optimal time corresponding to the _slowest_ task to complete.

| Number of consecutive tasks | Expected Seq. time to complete | Optimal time |
|-----------------------------|--------------------------------|--------------|
| 1                           | 1s                             | 1s           |
| 10                          | 55s      (~1 min)              | 10s          |
| __30__                      | __465s     (7 min 45s)__       | __30s__      |
| 100                         | 5050s    (1h 24 min 10s)       | 100s         |


The objective is of course to optimize the time required to conclude


### Single task run (interactive)

Let's first test this command in an interactive jobs:

```bash
### Access to ULHPC cluster (if not yet done)
(laptop)$> ssh iris-cluster
### Have an interactive job
# ... either directly
(access)$> si
# ... or using the HPC School reservation 'hpcschool'if needed  - use 'sinfo -T' to check if active and its name
# (access)$> srun --reservation=hpcschool --pty bash
(node)$>
```

In another terminal (or another screen tab/windows), connect to that job and run `htop`

``` bash
# check your running job
(access)$> sq
# squeue -u $(whoami)
   JOBID PARTIT       QOS                 NAME       USER NODE  CPUS ST         TIME    TIME_LEFT PRIORITY NODELIST(REASON)
 2171206  [...]
# Connect to your running job, identified by its Job ID
(access)$> sjoin 2171206     # /!\ ADAPT job ID accordingly, use <TAB> to have it autocatically completed
(node)$> htop # view of all processes
#               F5: tree view
#               u <name>: filter by process of <name>
#               q: quit
```
Note that you have **only one core** reserved by default in a interactive job: you may thus see activities on the other cores via `htop`, they are corresponding to processes run by the other users. Note that you can use the `--exclusive`  flag upon reservation to request en exclusive access to a full node.

Now execute the `stressme` command in the first terminal/windows to see its effect on the processor load:

``` bash
# Run directly
(node)$> ~/git/github.com/ULHPC/tutorials/sequential/basics/scripts/stressme
#  /usr/bin/stress -c 1 -t 20
stress: info: [59918] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd
stress: info: [59918] successful run completed in 20s

# Now let's create a convenient path to the scripts under the dedicated directory
# used for this session
(node)$> cd ~/tutorials/sequential
# create a symbolic link to the script directory
(node)$> ln -s ref.d/scripts .    # <-- don't forget the trailing '.'  means 'here'
(node)$> ls scripts/stressme      # should not fail
scripts/stressme
(node)$> ./scripts/stressme
stress: info: [59918] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd
stress: info: [59918] successful run completed in 20s
# Exit the job
(node)$> exit    # OR CTRL-D
(access)$>
```

Exit `htop` in its terminal (press 'q' to exit) and press `CTRL-D` to disconnect to return to the access server.
Quit the interactive job by pressing `CTRL-D` to disconnect to return to the access server.


## A First launcher (1 job, 1 task on one core)

We will create a first "_launcher_ script", responsible to perform a single task over 1 core.
We will copy the default template provided by the ULHPC team for such usage, [`launcher.serial.sh`](scripts/launcher.serial.sh).

```bash
(access)$> cd ~/tutorials/sequential # if not yet done
(access)$> cp ~/git/github.com/ULHPC/tutorials/sequential/basics/scripts/launcher.serial.sh .   # <- trailing '.' means 'here
(access)$> mv launcher.serial.sh launcher.stressme-serial.sh
```

Use your favorite editor (`nano`, `vim` etc) to edit it as follows:

```diff
--- ~/git/github.com/ULHPC/tutorials/sequential/basics/scripts/launcher.stressme-serial.sh 2020-12-10 16:25:24.564580000 +0100
+++ launcher.stressme-serial.sh 2020-12-10 17:02:16.847752000 +0100
 #! /bin/bash -l
 ############################################################################
 # Default launcher for serial (one core) tasks
 ############################################################################
-###SBATCH -J Serial-jobname
+#SBATCH -J StressMe-single
@@ -32,7 +32,7 @@
 # /!\ ADAPT TASK variable accordingly
 # Absolute path to the (serial) task to be executed i.e. your favorite
 # Java/C/C++/Ruby/Perl/Python/R/whatever program to be run
-TASK=${TASK:=${HOME}/bin/app.exe}
+TASK=${TASK:=${HOME}/tutorials/sequential/scripts/stressme}
```

Let's test it in an interactive job:

```bash
### Have an interactive job
# ... either directly
(access)$> si
# ... or using the HPC School reservation 'hpcschool'if needed  - use 'sinfo -T' to check if active and its name
# (access)$> srun --reservation=hpcschool --pty bash

# check usage
(node)$> ./launcher.stressme-serial.sh -h
NAME
  launcher.stressme-serial.sh: Generic launcher for the serial application
     Default TASK: $HOME/tutorials/sequential/scripts/stressme
USAGE
  [sbatch] ./launcher.stressme-serial.sh [-n]
  TASK=/path/to/app.exe [sbatch] ./launcher.stressme-serial.sh [-n]
OPTIONS:
  -n --dry-run: Dry run mode

# dry-run
(node)$> ./launcher.stressme-serial.sh -n
### Starting timestamp (s): 1607616148
/home/users/<login>/tutorials/sequential/scripts/stressme
### Ending timestamp (s): 1607616148"
# Elapsed time (s): 0

# real test
(node)$> ./launcher.stressme-serial.sh
### Starting timestamp (s): 1607616167
#  /usr/bin/stress -c 1 -t 20
stress: info: [64361] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd
stress: info: [64361] successful run completed in 20s
### Ending timestamp (s): 1607616187"
# Elapsed time (s): 20
```

_Hint_: for the lazy persons, you can define on the fly the `TASK` variable as follows:

```bash
(node)$> TASK=$(pwd)/scripts/stressme ./scripts/launcher.serial.sh -n
### Starting timestamp (s): 1607616148
/home/users/<login>/tutorials/sequential/scripts/stressme
### Ending timestamp (s): 1607616148"
# Elapsed time (s): 0

(node)$> TASK=$(pwd)/scripts/stressme ./scripts/launcher.serial.sh
### Starting timestamp (s): 1607616500
#  /usr/bin/stress -c 1 -t 20
stress: info: [103707] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd
stress: info: [103707] successful run completed in 20s
### Ending timestamp (s): 1607616520"
# Elapsed time (s): 20
```

Quit your interactive job (exit or `CTRL+D`) and submit it as a passive job:

``` bash
(access)$> sbatch ./launcher.stressme-serial.sh
```

_Hint_: for the lazy persons, just run:

``` bash
(access)$> TASK=$(pwd)/scripts/stressme sbatch ./scripts/launcher.serial.sh
```

### StressMe Job Campaign: For loop on  `sbatch` (VERY BAD)

**Now the job campaign for N tasks (N=30) can be obtained by submitting N=30 jobs**:

``` bash
# /!\ ALWAYS echo your commands in a for loop before going real
(access)$> for i in $(seq 1 30); do echo sbatch ./launcher.stressme-serial.sh $i; done
sbatch ./launcher.stressme-serial.sh 1
sbatch ./launcher.stressme-serial.sh 2
sbatch ./launcher.stressme-serial.sh 3
[...]
sbatch ./launcher.stressme-serial.sh 28
sbatch ./launcher.stressme-serial.sh 29
sbatch ./launcher.stressme-serial.sh 30

# Go real
(access)$> for i in $(seq 1 30); do sbatch ./launcher.stressme-serial.sh $i; done
Submitted batch job 2171802
Submitted batch job 2171803
Submitted batch job 2171804
Submitted batch job 2171805
Submitted batch job 2171806
Submitted batch job 2171807
Submitted batch job 2171809
Submitted batch job 2171810
Submitted batch job 2171811
Submitted batch job 2171812
Submitted batch job 2171813
Submitted batch job 2171814
Submitted batch job 2171815
Submitted batch job 2171816
Submitted batch job 2171817
Submitted batch job 2171818
Submitted batch job 2171819
Submitted batch job 2171820
Submitted batch job 2171821
Submitted batch job 2171822
Submitted batch job 2171823
Submitted batch job 2171824
Submitted batch job 2171825
Submitted batch job 2171827
Submitted batch job 2171828
Submitted batch job 2171829
Submitted batch job 2171830
Submitted batch job 2171831
Submitted batch job 2171832
Submitted batch job 2171834
```

Use `sq` to check the status of your job in the queue:

``` bash
(access)$> sq   # squeue -u $(whoami)
   JOBID PARTIT       QOS                 NAME       USER NODE  CPUS ST         TIME    TIME_LEFT PRIORITY NODELIST(REASON)
 2171819  batch    normal      StressMe-single  <login>    1     1 CG         0:19        59:41    12678 iris-081
 2171820  batch    normal      StressMe-single  <login>    1     1 CG         0:19        59:41    12678 iris-149
 2171818  batch    normal      StressMe-single  <login>    1     1 CG         0:18        59:42    12678 iris-081
 2171831  batch    normal      StressMe-single  <login>    1     1  R         0:13        59:47    12678 iris-075
 2171832  batch    normal      StressMe-single  <login>    1     1  R         0:13        59:47    12678 iris-075
 2171834  batch    normal      StressMe-single  <login>    1     1  R         0:13        59:47    12678 iris-075
 2171830  batch    normal      StressMe-single  <login>    1     1  R         0:15        59:45    12678 iris-075
 2171825  batch    normal      StressMe-single  <login>    1     1  R         0:16        59:44    12678 iris-082
 2171827  batch    normal      StressMe-single  <login>    1     1  R         0:16        59:44    12678 iris-082
 2171828  batch    normal      StressMe-single  <login>    1     1  R         0:16        59:44    12678 iris-082
 2171829  batch    normal      StressMe-single  <login>    1     1  R         0:16        59:44    12678 iris-082
 2171823  batch    normal      StressMe-single  <login>    1     1  R         0:18        59:42    12678 iris-149
 2171824  batch    normal      StressMe-single  <login>    1     1  R         0:18        59:42    12678 iris-149
 2171821  batch    normal      StressMe-single  <login>    1     1  R         0:19        59:41    12678 iris-149
 2171822  batch    normal      StressMe-single  <login>    1     1  R         0:19        59:41    12678 iris-149
```

Check the past jobs statistics upon completion:

``` bash
(access)$> sacct -X --format User,JobID,partition%12,state,time,start,end,elapsed,nnodes,ncpus,nodelist
   User        JobID    Partition      State  Timelimit               Start                 End    Elapsed   NNodes      NCPUS        NodeList
------- ------------ ------------ ---------- ---------- ------------------- ------------------- ---------- -------- ---------- ---------------
<login> 2171802             batch  COMPLETED   01:00:00 2020-12-10T17:42:38 2020-12-10T17:42:39   00:00:01        1          1        iris-075
<login> 2171803             batch  COMPLETED   01:00:00 2020-12-10T17:42:41 2020-12-10T17:42:43   00:00:02        1          1        iris-075
<login> 2171804             batch  COMPLETED   01:00:00 2020-12-10T17:42:41 2020-12-10T17:42:44   00:00:03        1          1        iris-075
<login> 2171805             batch  COMPLETED   01:00:00 2020-12-10T17:42:41 2020-12-10T17:42:45   00:00:04        1          1        iris-075
<login> 2171806             batch  COMPLETED   01:00:00 2020-12-10T17:42:41 2020-12-10T17:42:47   00:00:06        1          1        iris-075
<login> 2171807             batch  COMPLETED   01:00:00 2020-12-10T17:42:41 2020-12-10T17:42:48   00:00:07        1          1        iris-075
<login> 2171809             batch  COMPLETED   01:00:00 2020-12-10T17:42:41 2020-12-10T17:42:49   00:00:08        1          1        iris-075
<login> 2171810             batch  COMPLETED   01:00:00 2020-12-10T17:42:41 2020-12-10T17:42:50   00:00:09        1          1        iris-075
<login> 2171811             batch  COMPLETED   01:00:00 2020-12-10T17:42:44 2020-12-10T17:42:54   00:00:10        1          1        iris-076
<login> 2171812             batch  COMPLETED   01:00:00 2020-12-10T17:42:44 2020-12-10T17:42:55   00:00:11        1          1        iris-076
<login> 2171813             batch  COMPLETED   01:00:00 2020-12-10T17:42:44 2020-12-10T17:42:56   00:00:12        1          1        iris-076
<login> 2171814             batch  COMPLETED   01:00:00 2020-12-10T17:42:44 2020-12-10T17:42:57   00:00:13        1          1        iris-076
<login> 2171815             batch  COMPLETED   01:00:00 2020-12-10T17:42:44 2020-12-10T17:42:58   00:00:14        1          1        iris-076
<login> 2171816             batch  COMPLETED   01:00:00 2020-12-10T17:42:45 2020-12-10T17:42:59   00:00:14        1          1        iris-081
<login> 2171817             batch  COMPLETED   01:00:00 2020-12-10T17:42:46 2020-12-10T17:43:01   00:00:15        1          1        iris-081
<login> 2171818             batch  COMPLETED   01:00:00 2020-12-10T17:42:46 2020-12-10T17:43:04   00:00:18        1          1        iris-081
<login> 2171819             batch  COMPLETED   01:00:00 2020-12-10T17:42:47 2020-12-10T17:43:06   00:00:19        1          1        iris-081
<login> 2171820             batch  COMPLETED   01:00:00 2020-12-10T17:42:47 2020-12-10T17:43:06   00:00:19        1          1        iris-149
<login> 2171821             batch  COMPLETED   01:00:00 2020-12-10T17:42:48 2020-12-10T17:43:09   00:00:21        1          1        iris-149
<login> 2171822             batch  COMPLETED   01:00:00 2020-12-10T17:42:48 2020-12-10T17:43:09   00:00:21        1          1        iris-149
<login> 2171823             batch  COMPLETED   01:00:00 2020-12-10T17:42:49 2020-12-10T17:43:11   00:00:22        1          1        iris-149
<login> 2171824             batch  COMPLETED   01:00:00 2020-12-10T17:42:49 2020-12-10T17:43:12   00:00:23        1          1        iris-149
<login> 2171825             batch  COMPLETED   01:00:00 2020-12-10T17:42:51 2020-12-10T17:43:14   00:00:23        1          1        iris-082
<login> 2171827             batch  COMPLETED   01:00:00 2020-12-10T17:42:51 2020-12-10T17:43:17   00:00:26        1          1        iris-082
<login> 2171828             batch  COMPLETED   01:00:00 2020-12-10T17:42:51 2020-12-10T17:43:17   00:00:26        1          1        iris-082
<login> 2171829             batch  COMPLETED   01:00:00 2020-12-10T17:42:51 2020-12-10T17:43:19   00:00:28        1          1        iris-082
<login> 2171830             batch  COMPLETED   01:00:00 2020-12-10T17:42:52 2020-12-10T17:43:19   00:00:27        1          1        iris-075
<login> 2171831             batch  COMPLETED   01:00:00 2020-12-10T17:42:54 2020-12-10T17:43:22   00:00:28        1          1        iris-075
<login> 2171832             batch  COMPLETED   01:00:00 2020-12-10T17:42:54 2020-12-10T17:43:23   00:00:29        1          1        iris-075
<login> 2171834             batch  COMPLETED   01:00:00 2020-12-10T17:42:54 2020-12-10T17:43:24   00:00:30        1          1        iris-075
```

As can be seen, the campaign was initiated at 17:42:38 to be completed for the latest job at 17:43:24: **it took thus 46s to be completed**. It's **90,1% improvement on the sequential completion time**.
Note that you can quickly get the maximum value of the 7th column with `sort -k 7 [-n]` as follows:

``` bash
(access)$> sacct -X --format User,JobID,partition%12,state,time,start,end,elapsed,nnodes,ncpus,nodelist --noheader | sort -k 7 | tail -n 2
```
Repeating the experience on **100** consecutive tests demonstrated a completion between 18:43:22 and 18:46:19, i.e. on **less than 3 minutes (2m 57s or 177s)**. It corresponds to a **96,5% improvement on the sequential completion time**.
The below table summarizes the results:

| #tasks | #jobs | Seq. time | Optimal | `for [...] sbatch [...]` | Seq. Time Decrease | #nodes         |
|--------|-------|-----------|---------|--------------------------|--------------------|----------------|
| 1      | 1     | 1s        | 1s      | 1s                       | 0%                 | 1 (max 1)      |
| 10     | 10    | 55s       | 10s     | 19s                      | 65%                | 3 (max 10)     |
| __30__ | 30    | __465s__  | __30s__ | __46s__                  | __90,1% __         | __5 (max 30)__ |
| 100    | 100s  | 5050s     | 100s    | 177s                     | 96,5%              | 10  (max 100)  |


This works of course but this is **generally against best-practices**:

* **to complete N (serial) tasks, you need to submit N jobs that could be spread on up to N different nodes**.
    - This induces an **non-necessary overload of the scheduler** for (generally) very short tasks
* Node coverage is sub-optimal
    - your serial jobs can be spread on **up to N different nodes**

Imagine expanding the job campaign to 1000 or 10000 test cases if not more, you risk to degrade significantly the HPC environment (the scheduler will likely have trouble to manage so many short-live jobs).
**We have thus limit the number of jobs allowed per user**  (see `sqos`). You **better regroup the tasks per node** to exploit their core configuration (28 cores on `iris`, 128 on `aion`).


### StressMe Job Campaign: job arrays (BAD)

Slurm support [Job Arrays](https://slurm.schedmd.com/job_array.html), a mechanism for submitting and managing collections of similar jobs quickly and easily;
All jobs must have the same initial options (e.g. size, time limit, etc.) and you can limit the numner
You just need to specify the array index values using the `--array` or `-a` option of the `sbatch` command. You **SHOULD** set the maximum number of simultaneously running tasks from the job array  using a "%" separator -- typically match the number of cores per node (28 on `iris`, 128 on `aion`). For example `--array=0-100%28" will limit the number of simultaneously running tasks from this job array to 28.
Job arrays will have several additional environment variable set:

* `$SLURM_ARRAY_JOB_ID`     will be set to the first job ID of the array
* `$SLURM_ARRAY_TASK_ID`    will be set to the job array index value.
* `$SLURM_ARRAY_TASK_COUNT` will be set to the number of tasks in the job array.

We will copy the previous launcher:

```bash
(access)$> cd ~/tutorials/sequential # if not yet done
(access)$> cp launcher.stressme-serial.sh launcher.stressme-jobarray.sh
```

Use your favorite editor (`nano`, `vim` etc) to edit it as follows:

```diff
@@ -14,7 +14,8 @@
 #SBATCH --ntasks-per-node 1
 #SBATCH -c 1                   # multithreading per task : -c --cpus-per-task <n> request
 #__________________________
-#SBATCH -o logs/%x-%j.out      # log goes into logs/<jobname>-<jobid>.out
+#SBATCH --array 1-9%5
+#SBATCH -o logs/%x-%A_%a.out   # log goes into logs/<jobname>-<masterID>_<taskID>_<jobid>.out
@@ -75,7 +74,7 @@
 start=$(date +%s)
 echo "### Starting timestamp (s): ${start}"

-${CMD_PREFIX} ${TASK} ${OPTS}
+${CMD_PREFIX} ${TASK} ${OPTS} ${SLURM_ARRAY_TASK_ID}
```

Now you can run the same job campaign as before with a single `sbatch` command:

``` bash
(access)$> sbatch --array 1-30%28 ./launcher.stressme-jobarray.sh
```

**The above command WON'T work**: massive job arrays campaign were run in the past that used to overwhelm the slurm controllers.
To avoid this behaviour to repeat, **we drastically reduce the capabilities of job arrays**:

``` bash
(access)$> scontrol show config | grep -i maxarray
```

_In short_, **Don't use job arrays!!!**: you can do better with [GNU Parallel](http://www.gnu.org/software/parallel/)


## A better launcher (1 job, 28/128 tasks per node)

Of course, you can aggregate the number of tasks within a single slurm launcher, start them in the background (i.e. as child processes by using the ampersand `&` after a Bash command, and the `wait` command:

``` bash
TASK=stressme
for i in {1..30}; do
    srun -n1 --exclusive -c 1 --cpu-bind=cores ${TASK} $i &
done
wait
```
The ampersand `&` spawns the command `srun -n1 --exclusive -c 1 --cpu-bind=cores ${TASK} $i`  in the background and allows the loop to continue to the next iteration without waiting for this sub-process to finish.
This approach is dangerous and the location of the `wait` command can be optimized to match the number of tasks per nodes, i.e. when `${SLURM_NTASKS_PER_NODE}` (if set) or `${SLURM_CPUS_ON_NODE}` (28 on iris) process have been forked. The code would be as follows:

``` bash
TASK=stressme
ncores=${SLURM_NTASKS_PER_NODE:-${SLURM_CPUS_ON_NODE:-28}}
For i in {1..30}; do
    srun -n1 --exclusive -c 1 --cpu-bind=cores ${TASK} $i &
    [[ $((i%ncores)) -eq 0 ]] && wait
done
wait
```

You can test it from the sample launcher [`launcher.serial-ampersand.sh`](scripts/launcher.serial-ampersand.sh).

```bash
(access)$> cd ~/tutorials/sequential # if not yet done
(access)$> cp ~/git/github.com/ULHPC/tutorials/sequential/basics/scripts/launcher.serial-ampersand.sh .   # <- trailing '.' means 'here
(access)$> mv launcher.serial-ampersand.sh launcher.stressme-serial-ampersand.sh
(access)$> TASK=$(pwd)/scripts/stressme sbatch ./launcher.stressme-serial-ampersand.sh
Submitted batch job 2172531
```

Check the duration of the job afterward:

``` bash
# See all job steps with slist <jobID>
(access)$> slist 2172531
# sacct -j 2172531 --format User,JobID,Jobname%30,partition,state,time,elapsed,MaxRss,MaxVMSize,nnodes,ncpus,nodelist,AveCPU,ConsumedEnergyRaw
     User        JobID                        JobName  Partition      State  Timelimit    Elapsed     MaxRSS  MaxVMSize   NNodes      NCPUS        NodeList     AveCPU ConsumedEnergyRaw
--------- ------------ ------------------------------ ---------- ---------- ---------- ---------- ---------- ---------- -------- ---------- --------------- ---------- -----------------
svarrette 2172531                  StressMe-ampersand      batch  COMPLETED   01:00:00   00:00:34                              1         28        iris-061                         5859
          2172531.bat+                          batch             COMPLETED              00:00:34      3976K    178800K        1         28        iris-061   00:00:00              5625
          2172531.ext+                         extern             COMPLETED              00:00:36          0    108056K        1         28        iris-061   00:00:00              5859
          2172531.0                          stressme             COMPLETED              00:00:04          0    248548K        1          1        iris-061   00:00:00               666
          2172531.1                          stressme             COMPLETED              00:00:09          0    248548K        1          1        iris-061   00:00:00              1790
          2172531.2                          stressme             COMPLETED              00:00:08          0    248548K        1          1        iris-061   00:00:00              1582
          2172531.3                          stressme             COMPLETED              00:00:10          0    248548K        1          1        iris-061   00:00:00              2013
          2172531.4                          stressme             COMPLETED              00:00:07          0    248548K        1          1        iris-061   00:00:00              1364
          2172531.5                          stressme             COMPLETED              00:00:05          0    248548K        1          1        iris-061   00:00:00               914
          2172531.6                          stressme             COMPLETED              00:00:11          0    248548K        1          1        iris-061   00:00:00              2210
          2172531.7                          stressme             COMPLETED              00:00:14          0    248548K        1          1        iris-061   00:00:00              2807
          2172531.8                          stressme             COMPLETED              00:00:03          0    248548K        1          1        iris-061   00:00:00               465
          2172531.9                          stressme             COMPLETED              00:00:13          0    248548K        1          1        iris-061   00:00:00              2620
          2172531.10                         stressme             COMPLETED              00:00:19          0    248548K        1          1        iris-061   00:00:00              3704
          2172531.11                         stressme             COMPLETED              00:00:17          0    248548K        1          1        iris-061   00:00:00              3361
          2172531.12                         stressme             COMPLETED              00:00:28          0    248548K        1          1        iris-061   00:00:00              5047
          2172531.13                         stressme             COMPLETED              00:00:02          0    248548K        1          1        iris-061   00:00:00               226
          2172531.14                         stressme             COMPLETED              00:00:23          0    248548K        1          1        iris-061   00:00:00              4342
          2172531.15                         stressme             COMPLETED              00:00:16          0    248548K        1          1        iris-061   00:00:00              3183
          2172531.16                         stressme             COMPLETED              00:00:15          0    248548K        1          1        iris-061   00:00:00              3001
          2172531.17                         stressme             COMPLETED              00:00:24          0    248548K        1          1        iris-061   00:00:00              4480
          2172531.18                         stressme             COMPLETED              00:00:12          0    248548K        1          1        iris-061   00:00:00              2427
          2172531.19                         stressme             COMPLETED              00:00:18          0    248548K        1          1        iris-061   00:00:00              3534
          2172531.20                         stressme             COMPLETED              00:00:06          0    248548K        1          1        iris-061   00:00:00              1167
          2172531.21                         stressme             COMPLETED              00:00:31          0    248548K        1          1        iris-061   00:00:00              5285
          2172531.22                         stressme             COMPLETED              00:00:27          0    248548K        1          1        iris-061   00:00:00              4900
          2172531.23                         stressme             COMPLETED              00:00:21          0    248548K        1          1        iris-061   00:00:00              4021
          2172531.24                         stressme             COMPLETED              00:00:22          0    248548K        1          1        iris-061   00:00:00              4177
          2172531.25                         stressme             COMPLETED              00:00:31          0    248548K        1          1        iris-061   00:00:00              5155
          2172531.26                         stressme             COMPLETED              00:00:25          0    248548K        1          1        iris-061   00:00:00              4611
          2172531.27                         stressme             COMPLETED              00:00:20          0    248548K        1          1        iris-061   00:00:00              3861
          2172531.28                         stressme             COMPLETED              00:00:32       392K    248548K        1          1        iris-061   00:00:29              5298
          2172531.29                         stressme             COMPLETED              00:00:25          0    248548K        1          1        iris-061   00:00:00              4575
#
# seff 2172531
#
Job ID: 2172531
Cluster: iris
User/Group: svarrette/clusterusers
State: COMPLETED (exit code 0)
Nodes: 1
Cores per node: 28
CPU Utilized: 00:07:46
CPU Efficiency: 48.95% of 00:15:52 core-walltime
Job Wall-clock time: 00:00:34
Memory Utilized: 3.88 MB
Memory Efficiency: 0.00% of 112.00 GB
```

You can get the aggregated statistics with `-X`:

``` bash
(access)$> slist 2172531 -X
# sacct -j 2172531 --format User,JobID,Jobname%30,partition,state,time,elapsed,MaxRss,MaxVMSize,nnodes,ncpus,nodelist,AveCPU,ConsumedEnergyRaw -X
     User        JobID                        JobName  Partition      State  Timelimit    Elapsed     MaxRSS  MaxVMSize   NNodes      NCPUS        NodeList     AveCPU ConsumedEnergyRaw
--------- ------------ ------------------------------ ---------- ---------- ---------- ---------- ---------- ---------- -------- ---------- --------------- ---------- -----------------
svarrette 2172531                  StressMe-ampersand      batch  COMPLETED   01:00:00   00:00:34                              1         28        iris-061                         5859
# [...]
```

Repeat with 100 tasks (aggreated within a single job):

``` bash
(access)$> TASK=$(pwd)/scripts/stressme sbatch ./launcher.stressme-serial-ampersand.sh --max 100
Submitted batch job 2172535
(access)$> TASK=$(pwd)/scripts/stressme sbatch ./launcher.stressme-serial-ampersand.sh --max 10
Submitted batch job 2172537
# sq [...]
# Once completed:
(access)$> slist 2172535 -X

(access)$> slist 2172537 -X



```