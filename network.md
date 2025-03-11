## Network performance cheatsheet

### 1. Pick correct cables and connectors
- Depending on the distance in your expected network, you should choose different cables:
  - For short distances (up to 100 meters) use twisted pair cables, usually copper shows best results. Put it simple - converting data to light takes more time than converting it to electricity, and it makes difference when you need to achieve minimum latency and count each microsecond.
  - For longer distances (up to 1000 meters and more) use fiber optic cables.
- Avoid using too many routers and switches, as each of them adds latency to the network.
- Avoid using any wireless connections, as they are less reliable and have higher latency.
- Sending data over network may involve up to 7 layers of OSI model, and reducing number of layers or using more efficient protocols may reduce latency.
- Each layer of OSI model has its own overhead and may be optimized:
  - HTTP/HTTPS: depending on specifics, app may use JSON, XML, Binary data. Also, app may be establishing new sessions for each request. All this may be optimized.
  - TCP/UDP: selection of those two may have differences in latency. TCP is more reliable, but it has more overhead. UDP is less reliable, but it has less overhead. Also, TCP has congestion control, which may be disabled if you are sure that your network is reliable. There is a QUIC protocol, which is an alternative to TCP and has faster connection establishment, but it is not widely supported yet and takes up more network resources.
  - IP: IPv4 has less overhead than IPv6, so if you are sure that you don't need IPv6, you may disable it.
  - Ethernet: Ethernet has different encryptions, such as 8b/10b, 64b/66b, 128b/130b. It means that for each 8 bits of data 2 bits are added as metadata, meaning that 20% of all traffic is metadata. 64b/66b has only 3% of metadata in traffic, 128b/130b - 1.5%.

### 2. Structure you network correctly
- When limiting network access, try to block traffic at the edge of the network, not at the core. This will reduce the number of packets that need to be processed by the core network devices.
- Network services that are not used should be disabled. This will reduce the number of open ports and the number of processes that can be attacked. To see which services are running, use:
  ```shell
  ~$ netstat -tulnp
  Active Internet connections (only servers)
  Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
  tcp        0      0```
- When possible, avoid firewalls
- When possible, do not use TCP/IP stack at all, e.g. Unix sockets are faster than TCP/IP sockets because of the lack of TCP/IP encapsulation overhead.

### 3. Make sure that CPU is in the most efficient state
- Linux allows you to configure network modes such as RSS (Receive Side Scaling), RPS (Receive Packet Steering), RFS (Receive Flow Steering), and XPS (Transmit Packet Steering). These modes optimize how packets are processed across CPU cores to improve performance in multicore systems. This method is useful when single CPU becomes a bottleneck in receiving packets. Here's how you can configure these modes:
  - RSS (Receive side scaling - distributes incoming network traffic to multiple hardware queues and maps each queue to a specific CPU core) Needs check if supported by the network card `ethtool -l <interface>` and then `ethtool -N eth0 rx on`
  - RPS (Receive Packet Steering - software-based alternative to RSS that distributes packet processing across CPU cores): `echo <cpu_mask> > /sys/class/net/eth0/queues/rx-0/rps_cpus`
  - RFS (Receive Flow Steering - directs packets to the CPU where the corresponding application thread is running): `echo <number_of_flows> > /proc/sys/net/core/rps_sock_flow_entries`
  - XPS (Transmit Packet Steering - controls which CPU core is used for transmitting packets): `echo <cpu_mask> > /sys/class/net/<interface>/queues/tx-<queue_id>/xps_cpus`
- For hardcore development: packets may avoid kernel by using Data Plane Development Kit (DPDK)
- For simple optimization rx and tx may be simply set on separate cores. There is no specific process or thread to move, the setup should be done via IRQs (interrupt requests). They are located at `/proc/interruptions` and can be moved to specific cores via `echo <core_id> > /proc/irq/<irq_number>/smp_affinity`. Also, this may be achieved as restricting of sending interrupts to specific cores using `irqbalance` utility:
  ```shell
  # /etc/default/irqbalance
  
  # restrict IRQs to CPU 0
  IRQBALANCE_BANNED_CPUS="00000001"
  
  # limit IRQs to CPUs 0 and 1
  IRQBALANCE_BANNED_CPUS="00000011"
  ```

### 4. Detect network problems
- Hidden problems may be seen with netstat:
  ```shell
  ~$ netstat -i
  Kernel Interface table
  Iface             MTU    RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
  bond0            1500 14471358      0 764061 0        321889      0      0      0 BMmRU
  bond1            1500 2049662787  23250 2049218374 0           110      0      0      0 BMmRU
  eth0             1500  7408090      0      0 0        321889      0      0      0 BMsRU
  eth1             1500  7063268      0      0 0             0      0      0      0 BMsRU
  eth2             1500 1024848096  11625 1024806231 0           110      0      0      0 BMsRU
  eth4             1500 1024827489  11625 1024857738 0             0      0      0      0 BMsRU
  lo              65536  1408960      0      0 0       1408960      0      0      0 LRU
  ```
  where RX-ERR is the number of received packets with errors, RX-DRP is the number of received packets dropped, TX-ERR is the number of transmitted packets with errors, and TX-DRP is the number of transmitted packets dropped. OVR means that the hardware buffer is full and the packet is dropped - efficient metric to track saturation (method USE).
- `netstat -s` shows more detailed statistics, including the number of packets received and transmitted, the number of errors, and the number of dropped packets. It also shows stats of SYN packets retried (may mean that the network is overloaded),  number of attempted connects with invalid protocol, etc.
- use nicstat for speed and utilization statistics:
  ```shell
  # nicstat -z 1
  Time Int rKB/s wKB/s rPk/s wPk/s rAvs wAvs %Util Sat
  01:20:58 eth0 0.07 0.00 0.95 0.02 79.43 64.81 0.00 0.00
  01:20:58 eth4 0.28 0.01 0.20 0.10 1451.3 80.11 0.00 0.00
  01:20:58 vlan123 0.00 0.00 0.00 0.02 42.00 64.81 0.00 0.00
  01:20:58 br0 0.00 0.00 0.00 0.00 42.00 42.07 0.00 0.00
  Time Int rKB/s wKB/s rPk/s wPk/s rAvs wAvs %Util Sat
  01:20:59 eth4 42376.0 974.5 28589.4 14002.1 1517.8 71.27 35.5 0.00
  Time Int rKB/s wKB/s rPk/s wPk/s rAvs wAvs %Util Sat
  01:21:00 eth0 0.05 0.00 1.00 0.00 56.00 0.00 0.00 0.00
  01:21:00 eth4 41834.7 977.9 28221.5 14058.3 1517.9 71.23 35.1 0.00
  Time Int rKB/s wKB/s rPk/s wPk/s rAvs wAvs %Util Sat
  01:21:01 eth4 42017.9 979.0 28345.0 14073.0 1517.9 71.24 35.2 0.00
  ```
- use `tcptop` to find most active connections


### 5. Make adjustments to system settings `/etc/sysctl.conf`
- See current settings: `sysctl -a | grep tcp`
- For networks with high throughput - 10GbE - you may want to increase buffer sizes: 
  ```shell
  net.core.rmem_max = 16777216
  net.core.wmem_max = 16777216
  ```
- To handle conections count increase you may want to increase number of connection queues:
  ```shell
  # queue of incoming connections, awaiting syn-ack handshake
  net.ipv4.tcp_max_syn_backlog = 4096
  # queue of connections waiting for accept()
  net.core.somaxconn = 1024
  ```
- To handle large queue on NIC you may want to increase its capacity:
  ```shell
  net.core.netdev_max_backlog = 10000
  ```
- Usually connections via Internet support MTU 1500, but for local network you may want to increase it to 6000-9000.
- Try Tuned project for automatic tuning of system settings: `# tuned-adm list` and `# tuned-adm profile network-throughput` will show available profiles and set the network throughput profile respectively.