## Processor performance cheatsheet

### 1. Pick correct processor for your task.

When tuning an application, which should have really low latencies, consider revising following params:
- Is it a high-end hardware piece? What year was it manufactured?
- How many physical cores are there? Cloud providers usually communicate vCores, which are not real cores, but are hardware threads. In case lowest latency is required, developers would need to place each thread on separate physical core, therefore number of cores should be as high as expected OS threads within an application + leave some cores for other OS components, such as network (2 cores - rx/tx), file system, etc.
- Is it server hardware? (e.g. Intel Xeon, AMD EPYC - are server side lines of products). Does it have enough performance cores (P-cores)?
- Does the processor support multithreading? It is useful when turn on when more parallelism is needed, but should be turned off when each physical core is expected to produce the lowest latency + not many app threads are created.
- Does processor have big L-caches? L-caches are used to increase core performance, storing there data and avoiding unnecessary memory access.
- Does the whole system has good cooling? The thing is - overheating can lead literally to clock speed reduction (or throttling), specific error codes can be monitored.
- Which bus configuration does the processor have? Ideally all modern processors should have interconnections between each core (vs older models, when single central bus is used).
- Does the processor have support for SIMD istructions? Developers may utilize such functionality to speed up some math calculations.
- Does the processor support Error Correction codes? it is crucial for monitoring and fixing errors.

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


### 2. Analyze running app

There are few MUST-have steps when monitoring application performance:
- Usage and saturation stats: basically `htop` or `top` are enough in most cases 
to see % of CPU used. More concise info can be received using `mpstat -P ALL 1`
or `pidstat 1` - only CPU data is shown, no memory/network/os data. 
The hidden issue is that load can spike for less than a second,
therefore later other instruments may be tried: `perf` flame stats or opensource Netflix tool FlameScope.
Low IPC (instructions per cycle) - 0.2 or below - may indicate that CPU is not fully utilized: `perf stat -a -- sleep 10`.
- Use `USE` method to analyze app performance: Utilization, Saturation, Errors.
  - First of all, check Errors: `dmesg -T | tail ` or look for a particular error: `dmesg -T | grep -i "cpu clock throttled"`.
  - Utilisation of CPU is measured by % of time CPU is busy - can be monitored by `top`, `mpstat` or `pidstat`.
  - Saturation - for CPU usually load average is considered. It may be monitored by `uptime` or `top`. It has 3 values: 1, 5, 15 minutes.
  Growing 1 -> 5 -> 15 minutes may indicate that CPU is overloaded and the trend doesn't stop.
  Helpful info can be received as `car /proc/pressure/cpu` - it shows how many tasks are waiting for CPU.
  Should be interpreted as: 1.0 means 1 core is fully utilized, 2.0 - 2 cores are fully utilized, etc.
- Profiling: 
  - Start with app profiling, usually developers integrate prometheus or other monitoring tools to get metrics,
  usually it comes with Grafana dashboards, which are easy to read.
  - Use `profile` and `perf` tools for deeper analysis. Simplest way is to convert results into flame graph and then interpret it.
- Ask yourself questions about static enhancements, such as:
  - Are all cores being used and turned on? Is it physical or virtual core?
  - Is the task using resources which are expected? e.g. is it using GPU, when it should use CPU, or vice versa?
  - Are there any unnecessary threads running?
  - Is turbo boost enabled? It can be turned off in BIOS. Are there any power saving options enabled?
  - Are there any external constraints on CPU, such as burstable CPU in cloud?
  - is compiler configured for producing optimized code? Are there any compiler flags used? e.g. in rust it is `--release` flag, cpu is set to `native` by default.
- For rare cases when you need to achieve minimum possible latency, you can
  - Turn off hyper-threading and all other virtualizations in BIOS.
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
    This command will bind process with PID 10790 to CPUs from 7th to 10th.
  - Set process priority to make sure your app is not interrupted by other processes. use `chrt` command to set real-time priority.
  - Move auxiliary processes to other cores to avoid interference with your app. E.g. network, file system, etc. Rule of thumb for lowest latency is to have 1 physical core for each thread. This step would require:
    - Identifying which processes are running on which cores
    - Moving processes to other cores using `taskset` command.
  - Enable NUMA to optimize memory access. To check if NUMA is available on the machine and is enabled run `numactl --hardware`

### 3. Use tools

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
В первой строке должна выводиться обобщенная информация с момента загрузки.
Но в Linux вывод в столбцах procs и memory начинается со значений, характери-
зующих текущее состояние. (Возможно, когда-то это исправят.) С процессором
связаны столбцы:
- r: длина очереди на выполнение — общее количество потоков, готовых к выполнению.
- us: процент времени выполнения в режиме пользователя.
- sy: процент времени выполнения в режиме системы (ядра).
- id: процент времени бездействия.
- wa: процент времени, проведенного потоками в ожидании завершения ввода/вывода, когда потоки оставались заблокированными на время дискового ввода/
вывода.
- st: процент заимствованного времени — время, потраченное на обслуживание других клиентов в виртуализированных средах.

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

