## File system performance cheatsheet

#### 1. Know basics of what filesystem does
- stores persistent data on disk
- separates physical operations of reading/writing from same logical operations, e.g. creating a buffer and then flushing it to disk at once.
- interacts with memory and other caches to simplify access to data
- can have cache of different levels, where 1st may be RAM, 2nd - SSD, 3rd - flash memory, etc.
- has POSIX API for user programs to interact with it
- on Linux no matter what actual filesystem is used (ext3, ext4, FAT, NTFS, ZFS), it is shown as Virtual File System (VFS) to user programs.
- has random and sequential I/O (input/output), difference - random operations start from different places on disk, sequential - from the same place, which makes them faster (analogous to consecutive iterations over an array vs random access to elements).
- can guess whether next bytes will be read, and prefetch them from disk to cache when actual data is not requested yet (read-ahead operation).
- as database has WAL (Write-Ahead Logging), file system has journaling, which logs all operations before they are actually performed, so in case of crash, it can recover the state of filesystem.
- has actual processes running and occupying CPU and memory, so all affinity techniques and CPU optimizations are applicable here as well to each process.
- IOPS (input/output per second) don't mean much without knowing the size of data being read/written. For example, 1000 IOPS with 1MB block size is 1GB/s, but with 4KB block size is only 4MB/s.

#### 2. Tune filesystem for better performance
1. Analyze disks
   - pick NVMe SSDs for best performance in terms of speed and latency. Pick HDDs for best prices and capacity. Check how disk is connected: SATA, SAS, NVMe, PCIe, etc. Best performance is achieved with NVMe over PCIe (SATA 0.5 GB/s, PCIe >10 GB/s).
   - check disk speed and load:
     ```shell
        host:~# iostat
        Linux 6.8.0-45-generic (ubuntu-DE-Frankfurt-1gb) 	10/29/24 	_x86_64_	(1 CPU)
        
        avg-cpu:  %user   %nice %system %iowait  %steal   %idle
                   0.77    0.02    0.76    0.02    0.03   98.40
        
        Device             tps    kB_read/s    kB_wrtn/s    kB_dscd/s    kB_read    kB_wrtn    kB_dscd
        loop0             0.00         0.00         0.00         0.00         14          0          0
        vda               2.72         7.66        19.53        12.16   26613584   67839875   42252628
     ```
     where `tps` is transactions per second (IOPS), `kB_read/s` and `kB_wrtn/s` are read and write speeds, `kB_dscd/s` is discarded data speed, `kB_read`, `kB_wrtn`, `kB_dscd` are total read, write and discarded data.
   
     For additional stats use 
     ```shell
     host:~# iostat -sxz
     Linux 6.8.0-45-generic (ubuntu-DE-Frankfurt-1gb) 	10/29/24 	_x86_64_	(1 CPU)

     avg-cpu:  %user   %nice %system %iowait  %steal   %idle
                0.77    0.02    0.76    0.02    0.03   98.40

     Device             tps      kB/s    rqm/s   await  areq-sz  aqu-sz  %util
     loop0             0.00      0.00     0.00    0.00     1.27    0.00   0.00
     vda               2.73     39.39     0.64    0.48    14.45    0.00   0.02
     ```
     where  `rqm/s` is frequency of added and merged requests, `await` is time for average I/O operation including being in driver queue in milliseconds, `aqu-sz` - average queue size, `areq-sz` - average request size in kilobytes, `%util` - disk utilization.

      Additionally,  check pressure info with
      ```shell
      host:~# cat /proc/pressure/io
      some avg10=0.00 avg60=0.00 avg300=0.00 total=968813787
      full avg10=0.00 avg60=0.00 avg300=0.00 total=771437362
      ```
      It shows how much time system waited for I/O completion in percent. In my example - basically none.
    - when additional info is still needed about disk internal processes, which are not shown in OS, use `MegaCLI` for LSI controllers, `StorCLI` for Broadcom controllers, `PercCLI` for DELL controllers, etc. For example, those can show patrol reads, consistency checks, etc. `smartctl` can also show much useful info about disk health.
    - when not sure, which process is using disk, use `iotop` to see disk usage by process.
    - when using network solutions (NAS), check if MPIO (Multipath I/O) is used, and if not, check if it's available at all.

2. Analyze latencies:
   - when doing I/O operation, keep in mind 3 stages
     - duration of calls inside an application. Usually this level is enough to decide - whether synchronous code should be migrated to asynchronous, or it is better to use different library which supports different system calls, or maybe remove/restructure I/O operations.
     - duration of system calls: read(), pread64(), preadv(), preadv2(). When doing simple small sequential reads, read() is better than others, no additional overhead is implied. But when reading large files and resources are underutilized, pread/preadv2() is better, as it allows to read multiple blocks at once, and then process them in parallel.
     - duration of VFS calls: vfs_read(), vfs_write(). There is nothing much that can be done here, duration may be reduced either by rewriting an app to use another low-level API, or by using different filesystem under the hood.
   - check what is the bottleneck - CPU, disk, network, etc. For cases when the bottleneck is
     - CPU: check whether I/O operations are done in parallel, and if not, try to parallelize them. If possible, move processes to less loaded cores using `taskset` or set affinity inside an app.
     - disk: if disk is fully utilized, there is nothing much that can be done, except for adding more disks and distributing load between them. If disk is not fully utilized, check whether I/O operations are done in parallel, and if not, try to parallelize them. Research into differences between volumes and pools.
     - network: check whether network is fully utilized, and if not, try to parallelize I/O operations.
3. Analyze and restructure load itself.
   - if multiple processes use same disk, consider mounting separate disks or filesystems for each process. This will help achieve only sequential I/O operations, and avoid random ones. Check what is actually mounted with filesystems: 
       ```shell
        $ mount
        devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
        tmpfs on /run type tmpfs (rw,nosuid,nodev,noexec,relatime,size=98476k,mode=755,inode64)
        /dev/vda1 on / type ext4 (rw,relatime,discard,errors=remount-ro,commit=30)
       ```
      where `realtime` means disabling updates of access time to files, `commit=30` means that data is flushed to disk every 30 seconds. If only 1 main filesystem disk is present, and multiple processes are using it, it might be a sign to move some to separate filesystems or disks.
   - is the disk full? consider moving some data to another disk or compressing it, otherwise write operations may be slow. When disk gets full, just selecting segment for new writes may require additional time.
   - are there outliers in I/O latencies? Filesystems may have hidden operations, such as backups, syncs, metadata updates, prefetches, etc.
4. Setup caching
   - cloud providers usually setup caches automatically, read here how AWS supports its FXs for OpenZFS: [link](https://docs.aws.amazon.com/fsx/latest/OpenZFSGuide/performance.html). 
   - to manually setup caches, use `zpool status` from `zfsutils-linux`. For other filesystems refer to their documentation.

#### 3. Follow rules of thumb
1. If system has more reading, add more caching. If system has more writing, add more disks.
2. If system works with batches, increase block size (+ related buffers) to increase total throughput and slow down each separate operation. If system works with small data, decrease block size (+ buffers) to reduce total throughput and speed up single operation.
3. For low-level disk operations use commands which do not transfer data as much as possible - e.g. for deleting `ATA TRIM` or `SCSI UNMAP`.
4. Similar to `nice` for CPU, `ionice` may be used for changing priorities of I/O access and its planning for different processes.
5. Perform benchmarks for your disk to understand its capabilities and limitations. Use `fio`, `ioping`. Do not forget to purge caches before running benchmarks, as they may affect results.

#### 4. Build physical disk structure to match the load
1. For critical data choose disks with built-in cache systems, better when it comes with separate battery. Research into differences between MLC, SLC, TLC, QLC technologies of memory cells before making final decision.
2. For best performance choose NVMe SSDs with RAID 0. It doesn't provide redundancy, but it is the fastest way to read/write data. When not loosing data is critical, use RAID 1 or RAID 10.
3. When RAID 1 (or 10) is too expensive, use RAID 5 or RAID 6. They provide redundancy and are cheaper because use not only half of space (as RAID 1/10 does), but may be slower. If using ZFS, research into RAID-Z schemas.
4. If using HDD, research into PMR (Perpendicular Magnetic Recording) and SMR (Shingled Magnetic Recording) technologies. PMR is faster, but SMR is cheaper. SMR is slower because of the way it writes data - it writes data in shingles, and when data is written, it is written over the previous data, which makes it slower. SMR is good for archiving data, when data is written once and read many times. PMR is good for databases, when data is written and read many times.

#### 5. Use tooling to analyze performance
1. Use `df -h` to check disk usage and available space.
2. Use `du -sh /path/to/directory` to check disk usage of a particular directory.
3. Use `slabtop -o` to see kernel caches stats:
   - `buffer_head`: stores metadata about buffers - main objects for buffer I/O operations;
   - `dentry`: catalogues cache. Catalog is basically a list of files and directories in a directory;
   - `inode_cache`: cache of index nodex - objects that store metadata about files and folders;
   - `ext3_inode_cache`: cache of index nodes for ext3;
   - `ext4_inode_cache`: cache of index nodes for ext4;
   - `xfs_inode`: cache of index nodes for xfs;
   - `btrfs_inode`: cache of index nodes for btrfs;
4. Use `strace -ttT -p <pid>` to look for system calls to file system (read, write, open, close, etc.) and their duration:
   ```shell
    $ strace -ttT -p <pid>
    [...]
    21:31:11.111111 read(9, "..."..., 65536) = 65536 <0.028225>
    21:31:11.211111 read(9, "..."..., 65536) = 65536 <0.000036>
   ```
   When looking to the output, the last part - is latency in seconds. The 1st completed in 28 milliseconds, the 2nd - in 36 microseconds. Having same size, it means the second probably hit cache. 
5. Use `fatrace` to see what files are being accessed on the filesystem.
6. Use `filetop` to see which files are being accessed the most.
7. Use `opensnoop -T` to see which files are being opened and closed.
8. Use `cachestat -T 1` to see basic cache stats:
    ```shell
    $ cachestat -T 1
    TIME     HITS MISSES DIRTIES HITRATIO BUFFERS_MB CACHED_MB
    21:00:48 586  0      1870    100.00%  208        775
    21:00:49 125  0      1775    100.00%  208        776
    21:00:50 113  0      1644    100.00%  208        776
    21:00:51 23   0      1389    100.00%  208        776
    21:00:52 134  0      1906    100.00%  208        777
    ```
   Hit ratio is the most important metric here, it shows how many requests were served from cache. To increase hit ratio, try to reduce memory allocated to the app.
9. Use `ext4dist 10 1` to see read/write/open/fsync latencies distributions.
10. Use `ext4slower` to see the slowest filesystem operations.