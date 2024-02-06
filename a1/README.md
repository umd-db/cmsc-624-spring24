## Assignment 1: Database System Process models

In this assignment, you will get some hands-on experience exploring the performance and architectural differences between database system process models. The main challenge of this assignment will be to understand the existing code that we are providing you. By reading through this code, we hope you will get a sense of some of the basic implementation differences between thread-based and process-based process models.

- Assignment 1: The latex template for your submission can be found in cmsc624-a1-sol-template. You can upload the template to [Overleaf](https://www.overleaf.com/).


### Overview

The amount of code you have to write for this assignment is quite small. In short, we are providing implementations of thread-per-worker process model \(see chapter two of the [Architecture of a Database System](https://dsf.berkeley.edu/papers/fntdb07-architecture.pdf) paper that is in the assigned reading for this semester\) over a simple database stored as an array in main memory. Your goal in this assignment is to implement the process-per-worker, process-pool and thread-pool variants of these process models. Even for these variants, we have written most of the code for you --- we have just left **the most interesting parts of the code left for you** to implement. However, if you do not understand the thread-per worker code that we have provided you, you will likely be unable to add the missing code \(even though it is small\) for other variants.

Aside from writing code, this assignment involves **running some performance experiments** and **answering some questions** we ask below. Please **don't** git clone this repository. Instead, we will create a private git repository for each student, and each student can commit changes and push them to this private repository. Our grading server will run your code regularly, generate the output files, and automatically commit them back to you. This will enable the code from each student to run on an identical server (with the same performance characteristics), so that students do not have to be concerned that any strange performance numbers they may see is the result of their particular choice of hardware. 


### Docker
This assignment assumes access to an x86-64 machine. For students who are using an M1 or M2 mac with ARM chips, we have provided a Docker image that you can use to run your code. The Docker image is based on Ubuntu 20.04 and contains all the necessary dependencies to run the code. You can find the details in the Dockerfile. You can build the Docker image by running the following command in the `docker` directory:

```
docker build --platform linux/amd64 -t "cmsc624-a1" .

docker run -v $PWD:/home/a1 -ti --platform linux/amd64 --cap-add=SYS_PTRACE --security-opt seccomp=unconfined  --name a1 cmsc624-a1:latest 
```

You can restart the container by running the following command:

```
docker start -i a1

```
If you have access to a x86-64 machine, you can ignore the Docker image and run the code directly on your machine. The docker image may lead to performance issue and hence we recommend you to run the code directly on your machine when possible.


### Execution models

The assignment uses Posix Threads to implement the thread-per-request and thread-pool models. This [Linux Posix Threads Tutorial](http://www.yolinux.com/TUTORIALS/LinuxTutorialPosixThreads.html) provides a good reference for programming with pthreads on Linux. The assignment codebase uses pthreads for managing threads' lifecycles \(thread creation and deletion\). Inter-thread synchronization is mediated via pthread **mutex lock** and **signal/wait**.

The process-per-request and process-pool models are implemented using Linux processes. Processes communicate with each other using shared memory segments via **mmap\(\)**. The provided database class implementation has a few examples of uses of **mmap\(\)**.

### Synchronization

This assignment uses mutexes to mediate inter-thread and inter-process communication. Concurrency is usually covered \(at least theoretically\) in the intro to systems programming courses. We recommend you read through Remzi's OS book \([Thread API](http://pages.cs.wisc.edu/~remzi/OSTEP/threads-api.pdf), [Condition Variables](http://pages.cs.wisc.edu/~remzi/OSTEP/threads-cv.pdf)\) for some background on pthread's mutex locks, and condition variables if you have not seen them in the past.

### Codebase

The assignment models database processing requests that manipulate several records. The database consists of an array of `1000-byte-sized` records. The database exports three important functions: **GetRecord**, **LockRecord**, and **UnlockRecord**. 

* GetRecord: returns a reference to a record. 
* LockRecord and UnlockRecord: serve as mutex-locks, giving the calling process or thread exclusive access to a particular record.

Each request updates several records in the database. In order to run a request, a process or thread calls a request's execute function in order to process it. `Launcher` is the base class from which each execution model will derive \(`ProcessPoolLauncher`, `ProcessLauncher`, `ThreadPoolLauncher`, and `ThreadLauncher`\). Each of these classes overrides the launcher's  **ExecuteRequest** function.

**Your task** is to complete the implementations of the `ProcessLauncher`, `ProcessPoolLauncher`, and `ThreadPoolLauncher` classes. Look for **"YOUR CODE HERE"** annotations in the comments in cpp files. We have provided implementations of the thread-per-request execution models in `ThreadLauncher.cc`. You should use these implementations as references while building your process, thread-pool, and process-pool implementations.


#### high\_priority\_txn
To help you better understand the process_pool model, we have implmented a set of `high_priority` transactions. These can be identified by invoking the `IsHighPriority()` function on the `Request` object. We will assign `high_priority` transactions only for the `process_pool` model. Your code need to use `Process0` to exclusively execute `high_priority` transactions. You may also assign `high_priority` transactions to other processes, but the performance of the `process_pool` model will be evaluated based on the throughput of `Process0` executing `high_priority` transactions. 


### Command-line Arguments

The provided CMake file will compile your executable into a directory `build/db`:

```shell
mkdir build && cd build
cmake ..
make -j8
make install 
```

The executable takes parameters `--exp_type`, `--max_outstanding`, and `--pool_size`. Each of these three parameters takes **integers** as arguments. 

> The command-line parsing code is not robust enough to deal with incorrectly specified parameter arguments!

#### max\_outstanding and pool\_size

The **pool-based** schemes \(thread- and process-pools\) take a `pool_size` parameter as input \(specified in their constructors\). The parameter is used to limit the maximum number of threads and processes available to execute requests. The **non-pool-based** schemes \(thread- and process-per-request\) take a `max_outstanding` parameter as input \(again, specified in their constructors\). This parameter is used to limit the maximum number of in-flight requests at any given time.

The `pool_size` and `max_outstanding` parameters above are specified as **command-line arguments**, and will be used to measure the relative performance of each of the four execution models.

#### exp\_type

`--exp_type` specifies the type of execution model.

* 0 - PROCESS\_POOL
* 1 - PROCESS\_PER\_REQUEST
* 2 - THREAD\_POOL
* 3 - THREAD\_PER\_REQUEST

> If you wish to test your process- or thread-pool implementations, specify the appropriate argument to `exp_type` and additionally specify a `pool_size` \(as an integer\).

> If you wish to test your process- or thread-per-request implementations, specify the appropriate argument to `exp_type` and additionally specify the number of `max_outstanding` requests.

### Experiments

The executable will run for about a minute, and measure the throughput \(requests per second\) of the specified execution model. **The measured throughput is written to a file automatically** --- `results.txt`.

Finally, you will **need to report** measurements for running requests under varying levels of conflicts between requests. Two requests conflict if they try to access the same record in the database. We have provided a `--contention` option to induce conflicts among transactions. To run an experiment under high contention \(lots of conflicts\), add the `--contention` flag as an argument to the executable. Experiments run under low contention \(few conflicts\) without the `--contention` flag.

If a pair of requests conflict, then one of the requests block while the other makes progress. Thus, under high contention, we would expect requests to block each other frequently. On the other hand, under low contention, the probability of two requests blocking due to conflict is very low.

### Test

We have also provided code to test each of your implementations. In order to run a test, run the appropriately configured execution model with the `--test` option. If provided with the `--test` option, the executable will check for correctness using a serial execution model. Running your code with the `--test` option will not measure throughput.

If you use your own test scripts, make sure to include `killall build/db` in those scripts, otherwise, the processes that were spun off by the assignment will result in hundreds of zombie processes on machines. Alternatively, we have provided test scripts \(`lowcontention.sh` and `highcontention.sh`\) that you can use to generate results.

### Profiling

`perf` is a useful tool for measuring where time is spent when running code and for finding the root cause of surprising performance results in an experiment.

[System Performance Tools: perf](http://daslab.seas.harvard.edu/classes/cs165/doc/sections/S8_perf.pdf) provides a good reference for profiling with perf on Linux.

#### Perf

You can collect the samples using `perf record`. This generates an output file called `perf.data`. 

```shell
sudo perf record -g  build/db --exp_type 3 --max_outstanding 1
```

That file can then be analyzed, possibly on another machine, using the `perf report` and `perf annotate` commands.

```shell
sudo perf report -n
```

#### Flame Graph
 
The flame graph, invented by Brendan D. Gregg, helps to visualize the stats generated by perf and stored in `perf.data`. To use it, you first clone the flame graph's repository from GitHub:

```shell
git clone https://github.com/brendangregg/FlameGraph.git
```

After that, you can issue the following command to generate a graph:

```shell
sudo perf script | FlameGraph/stackcollapse-perf.pl | FlameGraph/flamegraph.pl > process.svg
```

Finally, to evaluate the program's performance, drag and drop the created file (`process.svg`) into the Chrome browser.

> [This post](https://www.markhansen.co.nz/profiler-uis/) is a quick literature review of CPU profiler user interfaces available for visualising the output of the linux perf tool.

### Hints

The header files corresponding to the four execution models contain the appropriate structures for communication between the main launcher thread, and processes/threads used to execute requests.

The thread- and process-pools are managed via a linked list. Each process or thread has a corresponding `process_state` or `thread_state` struct \(`include/ProcessPoolLauncher.h` and `include/ThreadPoolLauncher.h`\). These structs are linked together into a pool, which is used to indicate whether or not a process is currently in use.

### Deliverables - Part 1

Compiled and working code for the process-pool, and thread-pool execution models. We will evaluate your code using the `--test` option described above. Each execution model will be tested at pool sizes of 1, 2, 4, 8, 16, 32, 64, 128.

- Passing these test cases is the only way to earn credit for your code. We will award no points for code that does not pass test cases.
- Please do not write a serial program to bypass the test. We will check your code manually.
- In order to get comparable performance, please comment out the debugging code, such as the print function.

> Process, thread-pool, and process-pool execution models receive 7, 10, and 15 points, respectively. Total 32 points.

### Deliverables - Part 2

For each execution model, report throughput while varying the pool size or the maximum outstanding requests parameter. You must report throughput for all four execution models \(process-per-request, thread-per-request, thread-pool, and process-pool\). Measure throughput at the following parameter values \[1, 2, 4, 8, 16, 32, 64, 128\]. As mentioned above, we have provided test scripts \(`lowcontention.sh` and `highcontention.sh`\) which varies all of these parameters for you.

Report these throughput measurements under both high contention and low contention in text files **high-contention.txt** and **low-contention.txt** respectively. Each line of the file should begin with **the name of the execution model**, and **the list of measured throughput values** \(in increasing order of pool size or max outstanding requests\). 

For example:

```text
process_pool 2000 3000 4000 5000 6000 7000 ...
```

Provide a brief explanation \(between three to four paragraphs\) for the throughput trends you observe in a separate file **{your name}-a1-sol.pdf** (please use [cmsc624-a1-sol-latex-template.zip]({{ "/assets/assignments/hw1/cmsc624-a1-sol-latex-template.zip" | relative_url }}){:target="\_blank"} we provided). In particular, explain the differences between each process model:

* Why do some models get higher throughput than other models? 
* For the proces-pool model, how does the pool size effect the throughput of `high_priority` transactions?
* Which process model is the fastest and why is it the fastest? 
* Which one is slowest and why is it slowest? Please use [perf](https://www.brendangregg.com/perf.html) to profile the process model and explain where the bottleneck is. For example, what are the names of the hotspot **kernel functions** and what are their purposes?
* Why are the ones in the middle slower/faster than the two at the extremes? 
* How and why does throughout change as the maximum number of outstanding requests or pool size change? 
* In addition, explain the differences between high-contention and low-contention experiments.

> 44 points for performance results / analysis.

### Deliverables - Part 3

Finally, answer each of the following questions briefly \(at most two paragraphs per question\).

Questions

* The process-pool implementation must copy a request into a process's request buffer. The process-per-request implementation, however, does not need to copy or pass requests between processes. Why?
* The process-pool implementation requires a request to be copied into a process-local buffer before the request can be executed. On the other hand, the thread-pool implementation can simply use a pointer to the appropriate request. Why?
* Inspect the data structure of the pool in the thread-pool and process-pool implementations. Why the code might cause a memory leak?

> 8 points for each response. Total 24 points.

### How to submit

Please submit all text files (`{your name}`-a1-sol.pdf, high-contention.txt and lowcontention.txt) discussed above to the assignment 1 link on [ELMS](https://elms.umd.edu/). You do not need to submit any code, since we have access to your repositories.
