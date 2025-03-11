## CPU performance cheatsheet

Step 0 in CPU tuning is knowing basics of how CPUs work. Here are some key points:
- CPU has different cores, each core may have multiple threads.
- CPUs have different levels of caches (listed in order of descending closeness to a core): registers, L1, L2, L3, RAM, disk, clouds, etc. The bigger the closest cache, the less CPU will have to wait for data to be fetched from slow memory.
- CPU on hardware level operates with instructions, which are executed in a pipeline. Each instruction has several stages: fetch, decode, execute, write back. Instructions may be processed in parallel, but only if they don't depend on each other.
- When developer writes code, it has to be converted to instructions the following way:
  1. High-level code is compiled to assembly code.
  2. Assembly code is compiled to machine code.
  3. Machine code (instructions) is executed by CPU.
- Developers may optimize code to be executed faster if they know what CPU is capable of. Example - SIMD instructions, which allow to process multiple data in parallel.
- Operating System (OS) has a queue for each core, where it places threads to be executed. Ideally threads should not be moved between cores, as it leads to cache misses.
-  CPU has 2 kinds of activities in terms of its origin:
  - User mode - when CPU executes user code, e.g. your app.
  - Kernel mode - when CPU executes OS code, e.g. system calls, interruptions, network, file system, etc.
- CPU has interruptions - signals which stop current thread execution and switch to another thread:
  - hardware-level: when hardware sends a signal to CPU, e.g. network card has received a packet.
  - software-level: when OS sends a signal to CPU, e.g. timer has expired.
- CPU queues have preemption, so thread with higher priority may be executed first. Prioritizing of threads is important in OS concept. If thread is "glued" to specific core, it's called affinity.
- Queues of threads and its scheduling are a big deal when talking about high performance, and thereafter, it can be tuned to achieve the lowest latency.

### 1. Pick correct CPU for your task.

When tuning an application, which should have really low latencies, consider revising following params:
- Is it a high-end hardware piece? What year was it manufactured?
- How many physical cores are there? Cloud providers usually provide vCores, which are not real cores, but are hardware threads. In case the lowest latency is required, developers would need to place each thread on separate physical core, therefore number of cores should be as high as a number of expected OS threads within an application + leave some cores for other OS components, such as network (2 cores - rx/tx), file system, etc.
- Is it a server or a consumer hardware? (e.g. Intel Xeon, AMD EPYC - are server side lines of products). Does it have enough performance cores (P-cores)? Usually ARM architectures are optimised for energy efficient cores (E-cores), that's why they consume less power.
- Does the processor support multithreading? It is useful to turn on when more parallelism is needed (e.g. hundreds of OS threads are running simultaneously), but should be turned off when each physical core is expected to produce the lowest latency for only few processes.
- Does processor have big L-caches? L-caches are used to increase core performance, storing there data and avoiding unnecessary memory access.
- Does the whole system has good cooling? The thing is - overheating can lead to literal clock speed reduction (or throttling), specific error codes are produced when this happens.
- Which bus configuration does the processor have? Ideally all modern processors should have interconnections between each core (vs older models, where single central bus is used).
- Does the processor have support for SIMD instructions? Developers may utilize such functionality to speed up some math calculations.
- Does the processor support Error Correction codes? it is crucial for monitoring and fixing errors. E.g. those with throttling errors described above.

#### Below you can find examples of processor models, grouped by different 

| **Year of Production** | **Processor**           | **Manufacturer** | **L1 Cache**  | **L2 Cache** | **L3 Cache** | **Bus Specifications** | **Cores** | **Clock Speed**       | **Supported Instructions**                 |
|------------------------|-------------------------|------------------|---------------|--------------|--------------|------------------------|-----------|-----------------------|--------------------------------------------|
| 2006                   | Intel Core 2 Duo E6600   | Intel            | 64 KB         | 4 MB         | N/A          | 1066 MHz FSB            | 2         | 2.40 GHz              | MMX, SSE, SSE2, SSE3, SSSE3, Intel 64      |
| 2008                   | Intel Core i7-920        | Intel            | 64 KB         | 256 KB       | 8 MB         | 4.8 GT/s QPI            | 4         | 2.66 GHz              | SSE, SSE2, SSE3, SSSE3, SSE4.1, SSE4.2     |
| 2011                   | AMD FX-8150              | AMD              | 128 KB        | 8 MB         | N/A          | 5.2 GT/s HyperTransport | 8         | 3.60 GHz (4.20 GHz Turbo) | MMX, SSE, SSE2, SSE3, SSE4a, AVX           |
| 2013                   | Intel Core i5-4670K      | Intel            | 64 KB         | 256 KB       | 6 MB         | 5 GT/s DMI2             | 4         | 3.40 GHz (3.80 GHz Turbo) | MMX, SSE, SSE2, SSE3, SSE4.1, SSE4.2, AVX  |
| 2017                   | AMD Ryzen 7 1700X        | AMD              | 384 KB        | 4 MB         | 16 MB        | 8 GT/s HyperTransport   | 8         | 3.40 GHz (3.80 GHz Turbo) | MMX, SSE, SSE2, SSE3, SSE4a, AVX, AVX2     |
| 2018                   | Intel Core i9-9900K      | Intel            | 64 KB         | 256 KB       | 16 MB        | 8 GT/s DMI3             | 8         | 3.60 GHz (5.00 GHz Turbo) | MMX, SSE, SSE2, SSE3, SSE4.1, SSE4.2, AVX, AVX2 |
| 2020                   | AMD Ryzen 9 5900X        | AMD              | 384 KB        | 6 MB         | 64 MB        | PCIe 4.0                | 12        | 3.70 GHz (4.80 GHz Turbo) | MMX, SSE, SSE2, SSE3, SSE4a, AVX, AVX2, AVX-512 |
| 2021                   | Apple M1 Pro             | Apple            | 192 KB        | 24 MB        | N/A          | Integrated              | 10        | 3.20 GHz              | NEON, SIMD, AES, ARMv8.3-A, SVE            |
| 2022                   | Intel Core i9-13900K     | Intel            | 80 KB         | 2 MB         | 36 MB        | 16 GT/s DMI4            | 24        | 3.00 GHz (5.80 GHz Turbo) | MMX, SSE, SSE2, SSE3, SSE4.1, SSE4.2, AVX, AVX2, AVX-512 |


### 2. Analyze production apps

There are few MUST-have steps when monitoring application performance:
- Usage and saturation stats: basically `htop` or `top` are enough in most cases to see % of CPU used. More concise info can be received using `mpstat -P ALL 1` or `pidstat 1` - only CPU data is shown, no memory/network/OS data. 
The hidden issue is that load can spike for less than a second, therefore later other instruments may be tried: `perf` flame stats or opensource Netflix tool `FlameScope`.
Low IPC (instructions per cycle) - 0.2 or below - may indicate that CPU is not fully utilized, to see that param run `perf stat -a -- sleep 10`, needed param is `insn per cycle`:
```shell
user@host:~$ sudo perf stat -a -- sleep 10

 Performance counter stats for 'system wide':

         240141.18 msec cpu-clock                        #   24.001 CPUs utilized
            153290      context-switches                 #  638.333 /sec
             10163      cpu-migrations                   #   42.321 /sec
           2965744      page-faults                      #   12.350 K/sec
     1152840062816      cycles                           #    4.801 GHz
      375163392816      instructions                     #    0.33  insn per cycle
       80840641722      branches                         #  336.638 M/sec
         588897600      branch-misses                    #    0.73% of all branches
     6917040293634      slots                            #   28.804 G/sec
      541384600113      topdown-retiring                 #      7.8% Retiring
       91550076231      topdown-bad-spec                 #      1.3% Bad Speculation
       80247715660      topdown-fe-bound                 #      1.2% Frontend Bound
     6207248637252      topdown-be-bound                 #     89.7% Backend Bound
      231698120944      topdown-heavy-ops                #      3.3% Heavy Operations       #      4.5% Light Operations
       85898818200      topdown-br-mispredict            #      1.2% Branch Mispredict      #      0.1% Machine Clears
       39558725511      topdown-fetch-lat                #      0.6% Fetch Latency          #      0.6% Fetch Bandwidth
     1209354144211      topdown-mem-bound                #     17.5% Memory Bound           #     72.2% Core Bound

      10.005588691 seconds time elapsed
```
- Use `USE` method to analyze app performance: Utilization, Saturation, Errors.
  - First of all, check Errors: `dmesg -T | tail ` or look for a particular error: `dmesg -T | grep -i "cpu clock throttled"`.
  - Utilisation of CPU is measured by % of time CPU is busy - can be monitored by `top`, `mpstat` or `pidstat`.
  - Saturation - for CPU usually load average is considered. It may be monitored by `uptime` or `top`. It has 3 values: 1, 5, 15 minutes.
  Growing 1 -> 5 -> 15 minutes may indicate that CPU is overloaded and the trend doesn't stop. Should be interpreted as: 1.0 means 1 core is fully utilized, 2.0 - 2 cores are fully utilized, etc.
  
    Helpful info can be received from `cat /proc/pressure/cpu` - it shows how much time tasks are waiting for CPU:
    ```shell
    user@host:~$ cat /proc/pressure/cpu
    some avg10=0.90 avg60=15.42 avg300=20.96 total=10011765758
    full avg10=0.00 avg60=0.00 avg300=0.00 total=0
    ```
    Basically, `some` means that some tasks were waiting for CPU 0.9%, 15.42% and 20.96% of time in last 10, 60 and 300 seconds respectively. For similar pressure data for IO and memory `full` line will be non-zero, which means that all threads except idle are waiting for CPU.
  
- Profiling: 
  - Start with app profiling, usually developers add prometheus or other monitoring tools to an app to get metrics. Usually it comes with Grafana dashboards, which are easy to read.
  - Use `profile` and `perf` tools for deeper analysis. Simplest way is to convert results into flame graph and then interpret it.
- Ask yourself questions about static enhancements, such as:
  - Are all cores being used and turned on? Is it a physical or virtual core?
  - Is the task using resources which are expected to be used? e.g. is it using GPU, when it should use CPU, or vice versa?
  - Are there any unnecessary threads running?
  - Is turbo boost enabled? It can be turned off in BIOS. Are there any power saving options enabled?
  - Are there any external constraints on CPU, such as burstable CPU in cloud?
  - Is compiler configured for producing optimized code? Are there any compiler flags used? e.g. in rust it is `--release` flag, cpu is set to `native` by default.
- For rare cases when you need to achieve minimum possible latency, you can
  - Turn off hyper-threading and all other virtualization in BIOS.
  - Set P-states to the lowest value: P0/P1 to get maximum performance and use maximum power
  - Set C-states to the lowest value to restrict deep sleep of CPU.
  - Make sure T-states are used to save hardware from overheating. T-states are responsible for throttling, so when you overclock CPU, may be useful to throttle it to save hardware.
  - Same settings can be set in OS, e.g. in linux this would require commands like: 
    ```shell
    $ cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_governors
    performance powersave
    $ cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
    powersave
    ```
    This shows that current governor is set to `powersave`, which is not optimal for performance. To change it to `performance` use:
    ```shell
    $ echo performance > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
    ```
  - Set affinity for your app threads to specific cores to maximize cache usage and avoid cache misses and moving processes and data between cores. Affinity can be set within thread itself in code, e.g. in rust it would be something like: 
    ```rust
    use libc::syscall;
    use libc::SYS_gettid;
    use nix::sched::{sched_setaffinity, CpuSet};
    use nix::unistd::Pid;
  
    let pid: Pid = unsafe { Pid::from_raw(syscall(SYS_gettid) as i32) };
    let cpus: Vec<usize> = vec![0, 1, 2, 3];
  
    let mut cpu_set = CpuSet::new();
    for cpu in cpus {
      cpu_set.set(cpu).unwrap();
    }
    sched_setaffinity(pid, &cpu_set).unwrap();
    ```
    This is also available in linux shell:
    ```shell
    $ taskset -pc 7-10 10790
    pid 10790's current affinity list: 0-15
    pid 10790's new affinity list: 7-10
    ```
    This command will bind process with PID 10790 to CPUs from 7th to 10th. Of course, this will not work for languages with small control over runtime, e.g. Go manages goroutines itself and there's no reason to set affinity for them.
  - Set process priority to make sure your app is not interrupted by other processes. Use `chrt` command to set real-time priority.
  - Move auxiliary processes to different cores to avoid interference with your app threads. E.g. network, file system - all can be moved to specific cores. The rule of thumb for lowest latency is to have 1 physical core per app thread. This step would require:
    - Identifying cores for each running process
    - Moving processes to different cores using `taskset` command. Ideal situation is to have your app threads running on [1,2,3] cores and you auxiliary processes on [4,5,6,etc.] cores.
  - Enable NUMA to optimize memory access. NUMA is a concept which allows different CPUs have different slots of memory on physical level, which leads to differences in latency for memory access. Another term for that is "memory locality". The rule of thumb - each CPU should be using the closest RAM stick (its NUMA node). To check if NUMA is available on the machine and is enabled run `numactl --hardware`

### 3. Some tooling and its interpretation

#### 3.1 vmstat

```shell
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
r b swpd free buff cache si so bi bo in cs us sy id wa st
15 0 0 451732 70588 866628 0 0 1 10 43 38 2 1 97 0 0
15 0 0 450968 70588 866628 0 0 0 612 1064 2969 72 28 0 0 0
15 0 0 450660 70588 866632 0 0 0 0 961 2932 72 28 0 0 0
15 0 0 450952 70588 866632 0 0 0 0 1015 3238 74 26 0 0 0
[...]
```

The first line contains the summary since boot. But in Linux, the output in the `procs` and `memory` columns starts with the current state. (Maybe they will fix it someday.) The columns related to the processor are:
- r: run queue length - the total number of threads ready to run.
- us: the percentage of time spent in user mode.
- sy: the percentage of time spent in system mode (kernel).
- id: the percentage of idle time.
- wa: the percentage of time spent by threads waiting for I/O to complete, when threads were blocked for disk I/O.
- st: the percentage of borrowed time - time spent serving other clients in virtualized environments.

#### 3.2 mpstat

```shell
$ mpstat -P ALL
06:33:15 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
06:33:15 PM  all    4.33    0.00    2.75    0.01    0.00    0.59    3.56    0.00    0.00   88.76
06:33:15 PM    0    4.27    0.00    2.76    0.01    0.00    1.37    3.52    0.00    0.00   88.07
06:33:15 PM    1    4.30    0.00    2.73    0.00    0.00    1.06    3.66    0.00    0.00   88.25
06:33:15 PM    2    4.33    0.00    2.75    0.00    0.00    0.59    3.57    0.00    0.00   88.76
06:33:15 PM    3    4.34    0.00    2.75    0.00    0.00    0.50    3.59    0.00    0.00   88.81
```

- CPU: logical processor identifier or `all` - in the summary line.
- %usr: time spent in user mode, excluding %nice.
- %nice: time spent in user mode for low-priority processes.
- %sys: time spent in system mode (kernel).
- %iowait: waiting for I/O to complete.
- %irq: CPU consumption by hardware interrupts.
- %soft: CPU consumption by software interrupts.
- %steal: time spent serving other clients.
- %guest: time spent in guest virtual machines.
- %gnice: time spent on low-priority guests.
- %idle: idle time.

Key columns: %usr, %sys, and %idle. They determine CPU consumption and show the ratio of user/kernel time. With them, you can determine "hot" CPUs - those running at 100% load (%usr + %sys), when the load on others is lower, which may be due to single-threaded applications or device interrupts.

