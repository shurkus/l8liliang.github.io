---
layout: article
tags: Linux
title: Linux Network Performance
mathjax: true
key: Linux
---

[Red Hat Enterprise Linux Network Performance Tuning Guide](https://access.redhat.com/sites/default/files/attachments/20150325_network_performance_tuning.pdf)
{:.info} 


[how to tune your 100G host](https://www.es.net/assets/Uploads/100G-Tuning-TechEx2016.tierney.pdf)
{:.info} 

[100g Network Adapter Tuning](https://srcc.stanford.edu/100g-network-adapter-tuning)
{:.info} 

## PACKET RECEPTION IN THE LINUX KERNEL

### What is the relationship of DMA ring buffer and TX/RX ring for a network card?
```
https://stackoverflow.com/questions/47450231/what-is-the-relationship-of-dma-ring-buffer-and-tx-rx-ring-for-a-network-card

Short Answer: These are the same.

In this article it says:
	A variant of the asynchronous approach is often seen with network cards. 
	These cards often expect to see a circular buffer (often called a DMA ring buffer) established in memory shared with the processor; 
	each incoming packet is placed in the next available buffer in the ring, and an interrupt is signaled. 
	The driver then passes the network packets to the rest of the kernel, and places a new DMA buffer in the ring.

The DMA ring allows the NIC to directly access the memory used by the software. 
The software (NIC's driver in the kernel case) is allocating memory for the rings and then mapping it as DMA memory, so the NIC would know it may access it. 
TX packets will be created in this memory by the software and will be read and transmitted by the NIC (usually after the software signals the NIC it should start transmitting). 
RX packets will be written to this memory by the NIC and will be read and processed by the software (usually after an interrupt is issued to signal there's work).

> so the ring buffer is allocated in memory rather than NIC storage, and NIC knows where the rings are. 
> As I know the ring format is hardware-specific and there is some register in NIC of ring location, so the NIC will just find the ring and read it according to its own format, then it will find the packets? 
Yes, the format of the data in the rings should be consistent with the API to the NIC. 
Each vendor usually have a PRM (Programmer Reference Manual) which explains how the data should be constrcuted. 

Packet Recevie Sequence:
1.Network Device Receives Frames and these frames are transferred to the DMA ring buffer.
2.Now After making this transfer an interrupt is raised to let the CPU know that the transfer has been made.
3.In the interrupt handler routine the CPU transfers the data from the DMA ring buffer to the CPU network input queue for later time.
4.Bottom Half of the handler routine is to process the packets from the CPU network input queue and pass it to the appropriate layers.

```

### What are the Hardware Rx/Tx Queue in Ethernet controller
```
The ring is the representation of the device RX/TX queue. It is used for data transfer between the kernel stack and the device.

https://stackoverflow.com/questions/58658739/what-are-the-hardware-rx-tx-queue-in-ethernet-controller

The rx/tx queues contain DMA descriptors for incoming and outgoing packets.

NICs expose multiple circular buffers called queues or rings to transfer packets. 
The simplest setup uses only one receive and one transmit queue. 
Multiple transmit queues are merged on the NIC, incoming traffic is split according to filters or a hashing algorithm if multiple receive queues are configured. 
Both receive and transmit rings work in a similar way: the driver programs a physical base address and the size of the ring. 
It then fills the memory area with DMA descriptors, i.e., pointers to physical addresses where the packet data is stored with some metadata. 
Sending and receiving packets is done by passing ownership of the DMA descriptors between driver and hardware via a head and a tail pointer. 
The driver controls the tail, the hardware the head. Both pointers are stored in device registers accessible via MMIO.
```

### The NIC ring buffer
```
Receive ring buffers are shared between the device driver and NIC. 
The card assigns a transmit(TX) and receive (RX) ring buffer. 
As the name implies, the ring buffer is a circular buffer where an overflow simply overwrites existing data. 
It should be noted that there are two ways to move data from the NIC to the kernel, hardware interrupts and software interrupts, also called SoftIRQs.

The RX ring buffer is used to store incoming packets until they can be processed by the device
driver. The device driver drains the RX ring, typically via SoftIRQs, which puts the incoming
packets into a kernel data structure called an sk_buff or “skb” to begin its journey through the
kernel and up to the application which owns the relevant socket. 

The TX ring buffer is used to hold outgoing packets which are destined for the wire.

These ring buffers reside at the bottom of the stack and are a crucial point at which packet drop
can occur, which in turn will adversely affect network performance.

The netdev_max_backlog is a queue within the Linux kernel where traffic is stored after reception from
the NIC, but before processing by the protocol stacks (IP, TCP, etc). There is one backlog queue per CPU
core. A given core's queue can grow automatically, containing a number of packets up to the maximum
specified by the netdev_max_backlog setting. The netif_receive_skb() kernel function will find the
corresponding CPU for a packet, and enqueue packets in that CPU's queue. If the queue for that processor
is full and already at maximum size, packets will be dropped.

ring buffers是被NIC和driver共享的。
网卡自身会分配rx ring buffers和tx ring buffers。
正像名字所说的，如果不能及时处理ring buffers中的报文（由driver拷贝到内核skb中），那么后面的报文会覆盖之前的报文。

RX ring buffuers是用来存储接收到的报文，等待driver处理（拷贝到内核skb中过）。

driver通常通过SoftIRQs真正的处理ring buffers中的报文，把它们拷贝到内核skb中(skb是上半部创建的，所以这个拷贝是硬中断做的)，
然后再由内核协议栈处理并送到应用程序。
在内核协议栈处理之前，这些报文会放到CPU队列中(也是硬中断把报文放入CPU队列的,等待软中断处理，当然软中断最终会把报文发送给内核协议栈处理)，
每个CPU有一个这样的队列。如果CPU队列满了，报文也会被drop掉。
可以通过设置netdev_max_backlog调整CPU队列长度。

TX ring buffers用来存储带等待发送到网线上的数据。

使用ethtool -g 查看网卡ring buffers设置
使用ethtool -G 设置网卡ring buffers

```


### Interrupts and Interrupt Handlers
```
Interrupts from the hardware are known as “top-half” interrupts. When a NIC receives incoming
data, it copies the data into kernel buffers using DMA. The NIC notifies the kernel of this data by 
raising a hard interrupt. These interrupts are processed by interrupt handlers which do minimal
work, as they have already interrupted another task and cannot be interrupted themselves. Hard
interrupts can be expensive in terms of CPU usage, especially when holding kernel locks.
The hard interrupt handler then leaves the majority of packet reception to a software interrupt, or
SoftIRQ, process which can be scheduled more fairly.

Hard interrupts can be seen in /proc/interrupts where each queue has an interrupt vector in
the 1st column assigned to it. These are initialized when the system boots or when the NIC device
driver module is loaded. Each RX and TX queue is assigned a unique vector, which informs the
interrupt handler as to which NIC/queue the interrupt is coming from. The columns represent the
number of incoming interrupts as a counter value:

硬中断也叫上半部中断。
当网卡收到报文后，会把报文通过DMA方式拷贝到ring buffers，然后网卡会触发一个硬中断。
硬中断只做少量的处理(分配skb，设置skb protocol，把报文放入CPU队列)，因为硬中断占用CPU资源比较多（会关闭CPU中断），尤其是使用kernel lock的时候。
所以硬中断会把主要工作留给软中断处理。

网卡的中断号在系统启动或者网卡加载的时候分配，每个rx queue和tx queue使用一个中断号。

rx queue和tx queue就是ring buffers，一个queue对应一个ring buffer，
每个queue对应一个中断号，这样就可以利用多个中断和多个CPU并发处理ring buffers的数据。
可以使用ethtool -L设置网卡queue数量。
Queues are allocated when the NIC driver module is loaded. In some cases, the number of
queues can be dynamically allocated online using the ethtool -L command.

```

### SoftIRQs
```
Also known as “bottom-half” interrupts, software interrupt requests (SoftIRQs) are kernel routines
which are scheduled to run at a time when other tasks will not be interrupted. The SoftIRQ's
purpose is to drain the network adapter receive ring buffers. These routines run in the form of
ksoftirqd/cpu-number processes and call driver-specific code functions. They can be seen
in process monitoring tools such as ps and top.

软中断又叫下半部中断，是一种kernel进程。他的目的是处理ring buffers中的数据。
```

### NAPI Polling
```
NAPI, or New API, was written to make processing packets of incoming cards more efficient.
Hard interrupts are expensive because they cannot be interrupted. Even with interrupt
coalescence (described later in more detail), the interrupt handler will monopolize a CPU core
completely. The design of NAPI allows the driver to go into a polling mode instead of being
hard-interrupted for every required packet receive.
Under normal operation, an initial hard interrupt or IRQ is raised, followed by a SoftIRQ handler
which polls the card using NAPI routines. The polling routine has a budget which determines the
CPU time the code is allowed. This is required to prevent SoftIRQs from monopolizing the CPU.
On completion, the kernel will exit the polling routine and re-arm, then the entire procedure will
repeat itself.

NAPI用来提升中断效率。它使用轮询取代“每个包都触发一个硬中断”
基本流程呢是，当一个初始硬中断被处罚之后，会自动触发一个软中断。
在软中断执行的过程中，驱动会poll the card using NAPI routines，为了控制轮询时间，每个routine会有一个budget。
可以通过参数net.core.netdev_budget=600控制这个budget大小。

```

### Network Protocol Stacks
```
Once traffic has been received from the NIC into the kernel, it is then processed by protocol
handlers such as Ethernet, ICMP, IPv4, IPv6, TCP, UDP, and SCTP.

Finally, the data is delivered to a socket buffer where an application can run a receive function,
moving the data from kernel space to userspace and ending the kernel's involvement in the
receive process.
```

## Packet egress in the Linux kernel
```
Another important aspect of the Linux kernel is network packet egress. Although simpler than the
ingress logic, the egress is still worth acknowledging. The process works when skbs are passed
down from the protocol layers through to the core kernel network routines. Each skb contains a
dev field which contains the address of the net_device which it will transmitted through:

int dev_queue_xmit(struct sk_buff *skb)
{
 struct net_device *dev = skb->dev; <--- here
 struct netdev_queue *txq;
 struct Qdisc *q;

It uses this field to route the skb to the correct device:

if (!dev_hard_start_xmit(skb, dev, txq)) {

Based on this device, execution will switch to the driver routines which process the skb and finally
copy the data to the NIC and then on the wire. The main tuning required here is the TX queueing
discipline (qdisc) queue, described later on. Some NICs can have more than one TX queue. 

The following is an example stack trace taken from a test system. In this case, traffic was going
via the loopback device but this could be any NIC module:

 0xffffffff813b0c20 : loopback_xmit+0x0/0xa0 [kernel]
 0xffffffff814603e4 : dev_hard_start_xmit+0x224/0x480 [kernel]
 0xffffffff8146087d : dev_queue_xmit+0x1bd/0x320 [kernel]
 0xffffffff8149a2f8 : ip_finish_output+0x148/0x310 [kernel]
 0xffffffff8149a578 : ip_output+0xb8/0xc0 [kernel]
 0xffffffff81499875 : ip_local_out+0x25/0x30 [kernel]
 0xffffffff81499d50 : ip_queue_xmit+0x190/0x420 [kernel]
 0xffffffff814af06e : tcp_transmit_skb+0x40e/0x7b0 [kernel]
 0xffffffff814b0ae9 : tcp_send_ack+0xd9/0x120 [kernel]
 0xffffffff814a7cde : __tcp_ack_snd_check+0x5e/0xa0 [kernel]
 0xffffffff814ad383 : tcp_rcv_established+0x273/0x7f0 [kernel]
 0xffffffff814b5873 : tcp_v4_do_rcv+0x2e3/0x490 [kernel]
 0xffffffff814b717a : tcp_v4_rcv+0x51a/0x900 [kernel]
 0xffffffff814943dd : ip_local_deliver_finish+0xdd/0x2d0 [kernel]
 0xffffffff81494668 : ip_local_deliver+0x98/0xa0 [kernel]
 0xffffffff81493b2d : ip_rcv_finish+0x12d/0x440 [kernel]
 0xffffffff814940b5 : ip_rcv+0x275/0x350 [kernel]
 0xffffffff8145b5db : __netif_receive_skb+0x4ab/0x750 [kernel]
 0xffffffff8145b91a : process_backlog+0x9a/0x100 [kernel]
 0xffffffff81460bd3 : net_rx_action+0x103/0x2f0 [kernel] 

用户通过系统调用把报文拷贝到内核skb。
协议栈把skb下发给kernel routines。
driver处理skb并最终把它拷贝到NIC，然后通过wire发送出去。

此处可调整的是TX queueing discipline(qdisc)和TX queue length
```


## Networking Tools
```
To properly diagnose a network performance problem, the following tools can be used:

1.netstat
A command-line utility which can print information about open network connections and protocol
stack statistics. It retrieves information about the networking subsystem from the /proc/net/
file system. These files include:
• /proc/net/dev (device information)
• /proc/net/tcp (TCP socket information)
• /proc/net/unix (Unix domain socket information)
For more information about netstat and its referenced files from /proc/net/, refer to the
netstat man page: man netstat.

2.dropwatch
A monitoring utility which monitors packets freed from memory by the kernel. For more
information, refer to the dropwatch man page: man dropwatch.

3.ip
A utility for managing and monitoring routes, devices, policy routing, and tunnels. For more
information, refer to the ip man page: man ip.

4.ethtool
A utility for displaying and changing NIC settings. For more information, refer to the ethtool
man page: man ethtool. 
```

## Identifying the bottleneck

### ethtool
```
Packet drops and overruns typically occur when the RX buffer on the NIC card cannot be drained
fast enough by the kernel. When the rate at which data is coming off the network exceeds that
rate at which the kernel is draining packets, the NIC then discards incoming packets once the NIC
buffer is full and increments a discard counter. The corresponding counter can be seen in
ethtool statistics. The main criteria here are interrupts and SoftIRQs, which respond to
hardware interrupts and receive traffic, then poll the card for traffic for the duration specified by
net.core.netdev_budget.

The correct method to observe packet loss at a hardware level is ethtool.

The exact counter varies from driver to driver;

# ethtool -S eth3
 rx_errors: 0
 tx_errors: 0
 rx_dropped: 0
 tx_dropped: 0
 rx_length_errors: 0
 rx_over_errors: 3295
 rx_crc_errors: 0
 rx_frame_errors: 0
 rx_fifo_errors: 3295
 rx_missed_errors: 3295

```

### various tools available to isolate a problem area
```
There are various tools available to isolate a problem area. Locate the bottleneck by investigating
the following points:
• The adapter firmware level
  - Observe drops in ethtool -S ethX statistics
• The adapter driver level
• The Linux kernel, IRQs or SoftIRQs
  - Check /proc/interrupts and /proc/net/softnet_stat
• The protocol layers IP, TCP, or UDP
  - Use netstat -s and look for error counters.

```

### Here are some common examples of bottlenecks
```
1. IRQs are not getting balanced correctly. 

enable irqbalance or set irq smp_affinity

2. See if any of the columns besides the 1st column of /proc/net/softnet_stat are increasing.

# cat /proc/net/softnet_stat
0073d76b 00000000 000049ae 00000000 00000000 00000000 00000000 00000000 00000000 00000000
000000d2 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
0000015c 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

3. Data is making it up to the socket buffer queue but not getting drained fast enough.
Monitor the ss -nmp command and look for full RX queues. Use the netstat -s
command and look for buffer pruning errors or UDP errors. The following example shows
UDP receive errors:

# netstat -su
Udp:
 4218 packets received
 111999 packet receive errors
 333 packets sent 

4. Increase the application's socket receive buffer or use buffer auto-tuning by not
specifying a socket buffer size in the application. Check whether the application calls setsockopt(SO_RCVBUF) as that will override the default socket buffer settings.

5. Application design is an important factor. Look at streamlining the application to make it
more efficient at reading data off the socket. One possible solution is to have separate
processes draining the socket queues using Inter-Process Communication (IPC) to
another process that does the background work like disk I/O.

6. Use multiple TCP streams. More streams are often more efficient at transferring data

7. Use larger TCP or UDP packet sizes. Each individual network packet has a certain
amount of overhead, such as headers. Sending data in larger contiguous blocks will
reduce that overhead. 

```

## Performance Tuning

### SoftIRQ Misses
```
If the SoftIRQs do not run for long enough, the rate of incoming data could exceed the kernel's
capability to drain the buffer fast enough. As a result, the NIC buffers will overflow and traffic will
be lost. Occasionally, it is necessary to increase the time that SoftIRQs are allowed to run on the
CPU. This is known as the netdev_budget. The default value of the budget is 300. This will
cause the SoftIRQ process to drain 300 messages from the NIC before getting off the CPU:

# sysctl net.core.netdev_budget
net.core.netdev_budget = 300

This value can be doubled if the 3rd column in /proc/net/softnet_stat is increasing, which
indicates that the SoftIRQ did not get enough CPU time. Small increments are normal and do not
require tuning.

This level of tuning is seldom required on a system with only gigabit interfaces. However, a
system passing upwards of 10Gbps may need this tunable increased.

# cat softnet_stat
0073d76b 00000000 000049ae 00000000 00000000 00000000 00000000 00000000 00000000 00000000
000000d2 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
0000015c 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
0000002a 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000

For example, tuning the value on this NIC from 300 to 600 will allow soft interrupts to run for
double the default CPU time:

# sysctl -w net.core.netdev_budget=600
```

### CPU Power States    
设置CPU在最大频率工作
```
sudo cpupower frequency-set --governor performance
```

### Tuned  
Tuned使用network-throughput profile
```
# yum -y install tuned
# service tuned start
# tuned-adm profile network-throughput
```

### numa  
```
NUMA architecture splits a subset of CPU, memory, and devices into different “nodes”, in effect
creating multiple small computers with a fast interconnect and common operating system. NUMA
systems need to be tuned differently to non-NUMA systems. For NUMA, the aim is to group all
interrupts from devices in a single node onto the CPU cores belonging to that node. The Red Hat
Enterprise Linux 6.5 version of irqbalance is NUMA-aware, allowing interrupts to balance only
to CPUs within a given NUMA node.

First, determine how many NUMA nodes a system has. This system has two NUMA nodes:
# ls -ld /sys/devices/system/node/node*
drwxr-xr-x. 3 root root 0 Aug 15 19:44 /sys/devices/system/node/node0
drwxr-xr-x. 3 root root 0 Aug 15 19:44 /sys/devices/system/node/node1

Determine which CPU cores belong to which NUMA node. On this system, node0 has 6 CPU
cores:
# cat /sys/devices/system/node/node0/cpulist
0-5
Node 1 has no CPUs.:
# cat /sys/devices/system/node/node1/cpulist

Checking the whether a PCIe network interface belongs to a specific NUMA node:
# cat /sys/class/net/<interface>/device/numa_node
For example:
# cat /sys/class/net/eth3/device/numa_node
1
This command will display the NUMA node number, interrupts for the device should be directed to
the NUMA node that the PCIe device belongs to.
This command may display -1 which indicates the hardware platform is not actually non-uniform
and the kernel is just emulating or “faking” NUMA, or a device is on a bus which does not have
any NUMA locality, such as a PCI bridge.
```

使用numactl把进程绑定到网卡所属numa
```
[root@dell-per740-05 ~]# cat server 
for port in 5201 5202 5203 5204 5205 5206 5207 5208 5209 5210 5211 5212 5213 5214 5215 5216 5217 5218 5219 5220;do
        #numactl --cpubind=1 --membind=1 iperf3 -s -D -p $port &> log$port &
        if((port%2));then
		numactl --cpubind=netdev:ens4f0 --membind=netdev:ens4f0 iperf3 -s -D -p $port -B 199.2.2.1 &> log$port &
	else
		numactl --cpubind=netdev:ens4f1 --membind=netdev:ens4f1 iperf3 -s -D -p $port -B 199.3.3.1 &> log$port &
	fi
        #iperf3 -s -D -p $port &> log$port &
done

[root@dell-per740-05 ~]# cat client
for port in 5201 5202 5203 5204 5205 5206 5207 5208 5209 5210;do
#for port in 5201 5202 5203 5204 5205 5206 5207 5208 5209 5210 5211 5212 5213 5214 5215 5216 5217 5218 5219 5220;do
        if((port%2));then
                numactl --cpubind=netdev:ens4f0 --membind=netdev:ens4f0 iperf3 -c 199.2.2.3 -f g -t 30 -P 1 -p $port &> log$port &
        else
                numactl --cpubind=netdev:ens4f1 --membind=netdev:ens4f1 iperf3 -c 199.3.3.3 -f g -t 30 -P 1 -p $port &> log$port &
        fi
        #numactl --cpubind=netdev:ens4f4 --membind=netdev:ens4f4 iperf3 -c 199.2.2.1 -t 30 -P 1 -p $port &> log$port &
        #iperf3 -c 199.2.2.2 -t 30 -P 1 -p $port &> log$port &
done

wait

total_bw=0
for port in 5201 5202 5203 5204 5205 5206 5207 5208 5209 5210;do
#for port in 5201 5202 5203 5204 5205 5206 5207 5208 5209 5210 5211 5212 5213 5214 5215 5216 5217 5218 5219 5220;do
        #bw=$(cat log$port | grep SUM.*receiver| awk '{print $6}')
        bw=$(cat log$port | grep receiver | egrep -o "[0-9.]+ Gbits/sec"| awk '{print $1}')
        echo stream$port: $bw
        total_bw=$(echo $total_bw+$bw | bc -l)
done

echo total: $total_bw


```

### irqbalance  
如果irqbalance效果不好，可以自己设置中断亲和
```
# cat /proc/irq/211/smp_affinity
00020000,00000000
# 从右向左，每一位代表一个cpu，cpu序号从0开始。比如0000,0020代表cpu5
# 修改 affinigy
[root@moon 44]# echo f0 > smp_affinity
[root@moon 44]# cat smp_affinity
000000f0

# cat /proc/irq/211/smp_affinity_list 
49
这里面是cpu的列表，通过这个文件修改比较方便
# echo 1024-1031 > smp_affinity_list

calculate_cpu_affinity_value()
{
	local cpu_index=$1
	local cpu_count=$(cat /proc/cpuinfo |grep processor|wc -l)
	#local cpu_count=64
	local array_affinity
	for i in `seq 0 $((cpu_count-1))`;do
		if((i==cpu_index));then
			array_affinity[$i]=1
		else
			array_affinity[$i]=0
		fi
	done
	
	let i=0
	affinity=""
	let segment=0
	while((i<(cpu_count-1)));do
		num0=${array_affinity[$i]}
		num1=${array_affinity[((i+1))]}
		num2=${array_affinity[((i+2))]}
		num3=${array_affinity[((i+3))]}
		[ -z "$num0" ] && num0=0
		[ -z "$num1" ] && num1=0
		[ -z "$num2" ] && num2=0
		[ -z "$num3" ] && num3=0
		value=$(echo "obase=16;ibase=2;$num3$num2$num1$num0"|bc)
		if ! ((segment%6)) && ! ((i==0));then
			affinity="${value},${affinity}"
		else
			affinity="${value}${affinity}"
		fi
		let i=i+4
		let segment++
	done
	echo $affinity

}

set_affinity()
{       
        local devname=$1
	local reverse=$2
        #for irq in `cat /proc/interrupts | grep $dev_name | awk -F: '{print $1}' | awk '{print $1}'`
        #do
        #       echo 1 > /proc/irq/$irq/smp_affinity
        #done
        bus_info=$(ethtool -i $devname|grep bus-info|awk -F": " '{print $2}')
        [ -z "$bus_info" ] && { echo "get bus_info failed for dev $devname";return 1; }
        echo "devname: $devname"
        echo "bus_info: $bus_info"
        
        #if [ $(GetDistroRelease) -ge 7 ];then
        #        systemctl start cpupower.service
        #fi
        #sleep 2
        #cpupower frequency-info --governors
        #if [[ $? == 0 ]] # if $? != 0, means that no available governors, if $? = 0, means has governors(/sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors)
        #then    
        #        cpupower -c all frequency-set -g performance
        #fi
        #if [ $(GetDistroRelease) -ge 7 ];then
        #        systemctl stop irqbalance
        #else    
        #        service irqbalance stop
        #fi
        
        systemctl stop irqbalance
        #ethtool -K $devname gro on gso on tso on tx on rx on
        
        irqs=$(cat /proc/interrupts | grep "$bus_info" | awk '{print $1}' | tr -d :)
        if [ -z "$irqs" ];then
                irqs=$(cat /proc/interrupts | grep "$devname-" | awk '{print $1}' | tr -d :)
        fi
        irq_count=$(echo $irqs|wc -w)
	cpu_count=$(cat /proc/cpuinfo |grep processor|wc -l)
	
	numa_node=$(cat /sys/class/net/$devname/device/numa_node)
	numa_node_cpus=$(lscpu|grep -i "NUMA node$numa_node"|awk '{print $NF}'|tr "," " ") 
	numa_node_cpus=($numa_node_cpus)
        #process_count=$(echo $numa_node_cpus|awk '{print NF}')
	numa_node_cpu_count=${#numa_node_cpus[@]}

        echo "irqs= $irqs"
        echo "irq_count= $irq_count"
        echo "cpu_count = $cpu_count"
        echo "numa_node_cpu_count = $numa_node_cpu_count"
        
	# reverse numa_node_cpus
	if [ "$reverse" == "yes" ];then
		i=$((numa_node_cpu_count-1))
		j=0
		while((i>=0));do
			tmp_numa_node_cpus[$j]=${numa_node_cpus[$i]}
			let i--
			let j++
		done

		i=$((numa_node_cpu_count-1))
		while((i>=0));do
			numa_node_cpus[$i]=${tmp_numa_node_cpus[$i]}
			let i--
		done
	fi

	cpu_index=1
        for irq in $irqs;
        do      
		if((cpu_index>=numa_node_cpu_count));then
			cpu_index=1
		fi
		affinity=$(calculate_cpu_affinity_value ${numa_node_cpus[$cpu_index]} 2>/dev/null)  
		echo $affinity > /proc/irq/$irq/smp_affinity
		let cpu_index++
        done
        
        for irq in $irqs;
        do      
                cat /proc/irq/$irq/smp_affinity
        done

}

```


### Ethernet Flow Control (a.k.a. Pause Frames)
```
To enable Flow Control:
# ethtool -A eth3 rx on
# ethtool -A eth3 tx on

To confirm Flow Control is enabled:
# ethtool -a eth3
Pause parameters for eth3:
Autonegotiate: off
RX: on
TX: on 
```

### Interrupt Coalescence (IC)
```
Interrupt coalescence refers to the amount of traffic that a network interface will receive, or time
that passes after receiving traffic, before issuing a hard interrupt. Interrupting too soon or too
frequently results in poor system performance, as the kernel stops (or “interrupts”) a running task
to handle the interrupt request from the hardware. Interrupting too late may result in traffic not
being taken off the NIC soon enough. More traffic may arrive, overwriting the previous traffic still
waiting to be received into the kernel, resulting in traffic loss.

Most modern NICs and drivers support IC, and many allow the driver to automatically moderate
the number of interrupts generated by the hardware. The IC settings usually comprise of 2 main
components, time and number of packets. Time being the number microseconds (u-secs) that the
NIC will wait before interrupting the kernel, and the number being the maximum number of packets 
allowed to be waiting in the receive buffer before interrupting the kernel.

A NIC's interrupt coalescence can be viewed using ethtool -c ethX command, and tuned
using the ethtool -C ethX command. Adaptive mode enables the card to auto-moderate the
IC. In adaptive mode, the driver will inspect traffic patterns and kernel receive patterns, and
estimate coalescing settings on-the-fly which aim to prevent packet loss. This is useful when
many small packets are being received. Higher interrupt coalescence favors bandwidth over
latency. A VOIP application (latency-sensitive) may require less coalescence than a file transfer
protocol (throughput-sensitive). Different brands and models of network interface cards have
different capabilities and default settings, so please refer to the manufacturer's documentation for
the adapter and driver.

On this system adaptive RX is enabled by default:

# ethtool -c eth3
Coalesce parameters for eth3:
Adaptive RX: on TX: off
stats-block-usecs: 0
sample-interval: 0
pkt-rate-low: 400000
pkt-rate-high: 450000
rx-usecs: 16
rx-frames: 44
rx-usecs-irq: 0
rx-frames-irq: 0

The following command turns adaptive IC off, and tells the adapter to interrupt the kernel
immediately upon reception of any traffic:

# ethtool -C eth3 adaptive-rx off rx-usecs 0 rx-frames 0

A realistic setting is to allow at least some packets to buffer in the NIC, and at least some time to
pass, before interrupting the kernel. Valid ranges may be from 1 to hundreds, depending on
system capabilities and traffic received.
```

### The Adapter Queue
```
The netdev_max_backlog is a queue within the Linux kernel where traffic is stored after reception from
the NIC, but before processing by the protocol stacks (IP, TCP, etc). There is one backlog queue per CPU
core. A given core's queue can grow automatically, containing a number of packets up to the maximum
specified by the netdev_max_backlog setting. The netif_receive_skb() kernel function will find the
corresponding CPU for a packet, and enqueue packets in that CPU's queue. If the queue for that processor
is full and already at maximum size, packets will be dropped.

To tune this setting, first determine whether the backlog needs increasing.

The /proc/net/softnet_stat file contains a counter in the 2rd column that is incremented when the
netdev backlog queue overflows. If this value is incrementing over time, then netdev_max_backlog needs
to be increased.

# sysctl -w net.core.netdev_max_backlog=X
```

### MultiQueue
```
ethtool -L
ethtool -l
ethtool [ FLAGS ] -L|--set-channels DEVNAME     Set Channels
       [ rx N ]
       [ tx N ]
       [ other N ]
       [ combined N ]

```

### Adapter RX and TX Buffer Tuning
```
Adapter buffer defaults are commonly set to a smaller size than the maximum. Often, increasing
the receive buffer size is alone enough to prevent packet drops, as it can allow the kernel slightly
more time to drain the buffer. As a result, this can prevent possible packet loss.

# ethtool -G eth3 rx 8192 tx 8192
```

### Adapter Transmit Queue Length
```
The transmit queue length value determines the number of packets that can be queued before
being transmitted. The default value of 1000 is usually adequate for today's high speed 10Gbps
or even 40Gbps networks. However, if the number transmit errors are increasing on the adapter,
consider doubling it. Use ip -s link to see if there are any drops on the TX queue for an
adapter.

# ip link
2: em1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UP
mode DEFAULT group default qlen 1000
 link/ether f4:ab:cd:1e:4c:c7 brd ff:ff:ff:ff:ff:ff
 RX: bytes packets errors dropped overrun mcast
 71017768832 60619524 0 0 0 1098117
 TX: bytes packets errors dropped carrier collsns
 10373833340 36960190 0 0 0 0

The queue length can be modified with the ip link command:

# ip link set dev em1 txqueuelen 2000
# ip link
2: em1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UP
mode DEFAULT group default qlen 2000
 link/ether f4:ab:cd:1e:4c:c7 brd ff:ff:ff:ff:ff:ff
```

### Module parameters
```
# modinfo mlx4_en
filename: /lib/modules/2.6.32-246.el6.x86_64/kernel/drivers/net/mlx4/mlx4_en.ko
version: 2.0 (Dec 2011)
license: Dual BSD/GPL
description: Mellanox ConnectX HCA Ethernet driver
author: Liran Liss, Yevgeny Petrilin
depends: mlx4_core
vermagic: 2.6.32-246.el6.x86_64 SMP mod_unload modversions
parm: inline_thold:treshold for using inline data (int)
parm: tcp_rss:Enable RSS for incomming TCP traffic or disabled (0) (uint)
parm: udp_rss:Enable RSS for incomming UDP traffic or disabled (0) (uint)
parm: pfctx:Priority based Flow Control policy on TX[7:0]. Per priority bit mask (uint)
parm: pfcrx:Priority based Flow Control policy on RX[7:0]. Per priority bit mask (uint)

# ls /sys/module/mlx4_en/parameters
inline_thold num_lro pfcrx pfctx rss_mask rss_xor tcp_rss udp_rss

# cat /sys/module/mlx4_en/parameters/udp_rss
1

Some drivers allow these values to be modified whilst loaded, but many values require the driver
module to be unloaded and reloaded to apply a module option.

# echo 'options mlx4_en udp_rss=0' >> /etc/modprobe.d/mlx4_en.conf
# modprobe -r mlx4_en
# modprobe mlx4_en

# modprobe -r mlx4_en
# modprobe mlx4_en udp_rss=0

In some cases, driver parameters can also be controlled via the ethtool command.
For example, the Intel Sourceforge igb driver has the interrupt moderation parameter
InterruptThrottleRate. The upstream Linux kernel driver and the Red Hat Enterprise Linux
driver do not expose this parameter via a module option. Instead, the same functionality can
instead be tuned via ethtool:
# ethtool -C ethX rx-usecs 1000
```

### Adapter Offloading
```
In order to reduce CPU load from the system, modern network adapters have offloading features
which move some network processing load onto the network interface card. For example, the
kernel can submit large (up to 64k) TCP segments to the NIC, which the NIC will then break down
into MTU-sized segments. This particular feature is called TCP Segmentation Offload (TSO).

Offloading features are often enabled by default. It is beyond the scope of this document to cover
every offloading feature in-depth, however, turning these features off is a good troubleshooting
step when a system is suffering from poor network performance and re-test. If there is an
performance improvement, ideally narrow the change to a specific offloading parameter, then
report this to Red Hat Global Support Services. It is desirable to have offloading enabled
wherever possible.

Offloading settings are managed by ethtool -K ethX. Common settings include:
• GRO: Generic Receive Offload
• LRO: Large Receive Offload
• TSO: TCP Segmentation Offload
• RX check-summing = Processing of receive data integrity
• TX check-summing = Processing of transmit data integrity (required for TSO)

```

### Jumbo Frames
```
ip link set $dev mtu 9000
```

### sysctl
```
cat <<-EOF > /etc/sysctl.d/100.conf
net.core.default_qdisc = fq

net.ipv4.tcp_congestion_control = BBR

# allow testing with 2GB buffers

net.core.rmem_max = 2147483647

net.core.wmem_max = 2147483647

# allow auto-tuning up to 2GB buffers

net.ipv4.tcp_rmem = 4096 87380 2147483647

net.ipv4.tcp_wmem = 4096 65536 2147483647

net.core.optmem_max = 10485760

net.core.netdev_budget=600

net.ipv4.tcp_timestamps = 1

net.ipv4.tcp_sack = 0

net.ipv4.tcp_window_scaling = 1

net.core.netdev_max_backlog = 2500000
EOF

sysctl -p /etc/sysctl.d/100.conf 
```

### RSS: Receive Side Scaling
```
RSS is supported by many common network interface cards. On reception of data, a NIC can
send data to multiple queues. Each queue can be serviced by a different CPU, allowing for
efficient data retrieval. The RSS acts as an API between the driver and the card firmware to
determine how packets are distributed across CPU cores, the idea being that multiple queues
directing traffic to different CPUs allows for faster throughput and lower latency. RSS controls
which receive queue gets any given packet, whether or not the card listens to specific unicast Ethernet
addresses, which multicast addresses it listens to, which queue pairs or Ethernet queues get copies of
multicast packets, etc.

RSS Considerations
• Does the driver allow the number of queues to be configured?
Some drivers will automatically generate the number of queues during boot depending on
hardware resources. For others it's configurable via ethtool -L.
• How many cores does the system have?
RSS should be configured so each queue goes to a different CPU core.
```

### RPS: Receive Packet Steering
```
Receive Packet Steering is a kernel-level software implementation of RSS. It resides the higher
layers of the network stack above the driver. RSS or RPS should be mutually exclusive. RPS is
disabled by default. RPS uses a 2-tuple or 4-tuple hash saved in the rxhash field of the packet
definition, which is used to determine the CPU queue which should process a given packet. 
```

## NIC Tuning Summary
```
• SoftIRQ misses (netdev budget)
• "tuned" tuning daemon
• "numad" NUMA daemon
• CPU power states
• Interrupt balancing 
• Pause frames
• Interrupt Coalescence
• Adapter queue (netdev backlog)
• Adapter RX and TX buffers
• Adapter TX queue
• Module parameters
• Adapter offloading
• Jumbo Frames
• TCP and UDP protocol tuning
• NUMA locality
```
