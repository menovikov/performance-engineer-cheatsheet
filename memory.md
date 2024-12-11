## Memory performance cheat sheet

### 1. Know how computer memory works
When talking about memory, sometimes only RAM is considered. However, memory should be understood as a virtual storage space that includes RAM and other cold storages. When OS or a program runs out of RAM, it starts using different storages like SSD, HDD or other appliances if configured. This is called swapping. Swapping is a slow process because SSD and HDD are slower than RAM. Swapping should be avoided whenever possible if maximum memory speed is needed.

To turn off swapping, use:
```shell
  $ swapoff -a
```
In case you need to experiment with swapping and it's not configured by default, you can create a swap file, e.g. for 2GB:
```shell
$ fallocate -l 2G /swapfile
$ chmod 600 /swapfile
$ mkswap /swapfile
$ swapon /swapfile
```
Check it with `swapon --show` or `htop`.

### 2. Choose the right memory type
- Higher DDR version means higher memory speed.
- It matters where memory stick is physically located. Most modern programs require high concurrency, so NUMA over UMA is usually preferred. NUMA allows each CPU to have its own memory, which is faster than having single memory bus for all CPUs.
- Memory channels are important. More channels mean more memory bandwidth. For example, Intel Core I7 supports up to 4 channels, which quadruples memory bandwidth compared to single-channel memory.

To find out physical characteristics of memory plugged into motherboard, use:
```shell
  $ dmidecode --type memory

    Getting SMBIOS data from sysfs.
    SMBIOS 3.5.0 present.
    
    Handle 0x001A, DMI type 16, 23 bytes
    Physical Memory Array
        Location: System Board Or Motherboard
        Use: System Memory
        Error Correction Type: Single-bit ECC
        Maximum Capacity: 2 TB
        Error Information Handle: Not Provided
        Number Of Devices: 8
    
    Handle 0x001B, DMI type 17, 92 bytes
    Memory Device
        Array Handle: 0x001A
        Error Information Handle: Not Provided
        Total Width: 80 bits
        Data Width: 64 bits
        Size: 32 GB
        Form Factor: DIMM
        Set: None
        Locator: CPU0_DIMM_A1
        Bank Locator: NODE 0
        Type: DDR5
        Type Detail: Synchronous Registered (Buffered)
        Speed: 4800 MT/s
        Manufacturer: <BAD INDEX>
        Serial Number: <BAD INDEX>
        Asset Tag: <BAD INDEX>
        Part Number: <BAD INDEX>
        Rank: 1
        Configured Memory Speed: 5414 MT/s
        Minimum Voltage: 1.1 V
        Maximum Voltage: 1.1 V
        Configured Voltage: 1.1 V
        Memory Technology: DRAM
        Memory Operating Mode Capability: Volatile memory
        Firmware Version: <BAD INDEX>
        Module Manufacturer ID: Bank 15, Hex 0xD6
        Module Product ID: 0xAD00
        Memory Subsystem Controller Manufacturer ID: Unknown
        Memory Subsystem Controller Product ID: Unknown
        Non-Volatile Size: None
        Volatile Size: 32 GB
        Cache Size: None
        Logical Size: None
```


### 3. Optimize memory usage
- Allocate memory as little as possible. Memory allocation is a slow process, so it is better to allocate memory once and then reuse it. Not it only avoids calls malloc() and free(), but under the hood reduces memory pages movement from list of free pages to list of used pages and vice versa.
- Use big pages. To set it up there are few ways:
  - Manually set up big pages in the OS:
  ```shell
    $ echo 50 > /proc/sys/vm/nr_hugepages
    $ grep Huge /proc/meminfo
    AnonHugePages: 0 kB
    HugePages_Total: 50
    HugePages_Free: 50
    HugePages_Rsvd: 0
    HugePages_Surp: 0
    Hugepagesize: 2048 kB
    ```
  - Find out whether OS supports Transparent Huge Pages (THP) and enable it.
- When compiling or running an app, choose memory allocator which suits your needs, e.g. to use libtcmalloc:
```shell
  $ export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libtcmalloc_minimal.so.4
```
When picking memory allocator, start by comparing default one with tcmalloc and jemalloc, as they are the most popular ones. Try running them on real-life scenarios, because each particular app may have different memory usage patterns.
- Use NUMA nodes. If an app can be allocated to a single NUMA node, it will be faster than if it is allocated to multiple nodes. Also that means using core affinity. To check NUMA nodes:
```shell
  $ numactl --hardware
  available: 2 nodes (0-1)
  node 0 cpus: 0 1 2 3
  node 0 size: 8192 MB
  node 0 free: 4096 MB
  node 1 cpus: 4 5 6 7
  node 1 size: 8192 MB
  node 1 free: 4096 MB
  node distances:
  node   0   1
    0:  10  20
    1:  20  10
```
Then to run an app on a specific NUMA node use:
```shell
  $ numactl --membind=0 ./my_app
```
or to move already running process use
```shell
  $ numactl --membind=0 <pid>
```

### 4. Monitor memory usage
- 4.1 Use different tools to analyze memory usage:
    - use `free -h` to check memory usage and swap usage. Low free memory and high swap usage indicate that more memory is needed. Keep in mind that buf/cache is not true used memory, it is used for caching and can be freed if needed.
    - use `top/htop` to find processes which consume the most memory.
    - `sar -B` to look for long page scans (columns pgscank/s and pgscand/s). Long periods above 10s indicate that the system is under memory pressure, more memory is needed.
    - check latencies for memory access with `cat /proc/pressure/memory`.
    - check OOM (Out Of Memory) errors in `dmesg -T | grep -i "Out of memory"`.
- 4.2 Look for memory leaks inside applications, which grow constantly. Simplest ways are profiling tools provided by programming languages, e.g. pprof for Go. It provides a heap profile, which shows memory usage and helps to find memory leaks.
- 4.3 If swapping is enabled and deeper memory analysis is needed, use `perf record` for profiling and `perf report` for analyzing. Some examples are:
  - `perf record -e page-faults -c 1 -p 1843 -g -- sleep 60` - to find all page faults for process 1843 - situations when accessed memory is not in RAM and is being uploaded from disk.
  - `perf mem record command` - track all attempts to access memory from a particular command.

