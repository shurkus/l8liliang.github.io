---
layout: article
tags: NIC
title: 网卡收发包流程
mathjax: true
key: python
---

## softnet_data数据结构
```
每个cpu都有收发包的入口队列和出口队列，他们都包含在一个sofnet_data结构体中。
每个cpu有一个softnet_data结构体。

struct softnet_data
{
	int     		throttle;
	int 	cng_level;
	int	avg_blog;
	struct 	sk_buff_head 	input_pkt_queue;
	struct 	list_head 	poll_list;
	struct 	net_device 	*output_queue;
	struct 	sk_buff 	*completion_queue;
	struct 	net_device 	backlog_dev;
}

throttle,cng_level,avg_blog 用于拥塞控制
input_pkt_queue 非NPAI的网卡收到的包都放到这个队列中
poll_list 是有数据要接收的设备列表
output_queue 是有数据要发送的设备列表
completion_queue 是需要释放的发送缓冲区列表
```

## 收包流程
```
https://stackoverflow.com/questions/43696997/ring-buffers-and-dma

Network Device Receives Frames and these frames are transferred to the DMA ring buffer.
Now After making this transfer an interrupt is raised to let the CPU know that the transfer has been made.
In the interrupt handler routine the CPU transfers the data from the DMA ring buffer to the CPU network input queue for later time.
Bottom Half of the handler routine is to process the packets from the CPU network input queue and pass it to the appropriate layers.

1、网卡收到数据包后，通过DMA把报文发送到ring buffer（DMA ring buffer = NIC RX/RXring buffer）
2、网卡给CPU发送一个硬件中断
3、中断处理程序，也就是驱动程序，根据包长度分配skb_buff，并且把包（帧）拷贝到skb_buff中。（如果设备使用DMA，驱动程序只需要初始化一个指针，不需要拷贝。如果不支持DMA就多一次copy？）
	a)如果是非NAPI设备，skb_buff会放到对应的CPU的softnet_data.input_pkt_queue中。所有的非NAPI网卡公用这个相同的入口队列。
	b)如果是NAPI设备，每个网卡有自己的队列。并且网卡的poll方法（该方法在net_device中定义，由下半部softirq调用）直接从网卡的环形缓冲区(内存映射？)取数据。
	c)input_pkt_queue队列的最大长度是300，如果超出这个长度，新的包就会被丢弃。
4、中断处理程序，对一些skb_buff字段初始化，以便上层网络使用，比如skb_protocol
5、中断处理程序，调度软中断NET_RX_SOFTIRQ，把自己加入cpu.softnet_data.poll_list中，并结束
6、当软中断NET_RX_SOFTIRQ被执行时，net_rx_action函数会取出cpu.softnet_data.poll_list中的第一个设备，执行该设备的poll方法（这里只讨论NAPI）
7、poll方法大概会调用netif_receive_skb方法
8、netif_receive_skb主要完成三个任务
	a)把包的副本发送给每个分流器
	b)把包的副本发送给skb->protocol所关联的协议处理函数
	c)一些其它功能，比如bridge，bonding
9、三层协议最后可以转发、丢弃、接收这个报文。

```

## 发包流程
```
1、调用dev_queue_xmit函数
2、如果没有qdisc队列规则，直接调用hard_start_xmit方法发送数据
3、如果有qidsc队列规则，调用qdisc_enqueue（把包加入网卡队列 内存映射？）->
	qidsc_run（不停的调用qdisc_restart，直到队列停止）->
	qdisc_restart（从队列取包）->
	hard_start_xmit（由网卡发送）
4、hard_start_xmit可能成功，也可能失败，这是因为当网卡的内存剩余不足一个mtu大小时，驱动会调用netif_stop_queue停止出口队列。
   后面当内存可用时，设备会产生一个中断，中断处理程序通过调用netif_wake_queue恢复出口队列。
   失败了也没关系，因为后续软中断会被触发，继续发送。
5、两种情形触发NET_TX_SOFTIRQ
   a)netif_wak_queue->_netif_schedule->raise_softirq（此时会把设备自己加入cpu.softnet_data.output_queue中）
   b)当发送完成，且驱动程序通过dev_kfree_skb_irq通知缓冲区可以释放时(此时会把缓冲区地址加入cpu.softnet_data.completion_queue中)
6、当NET_TX_SOFTIRQ执行时，完成两个工作
   a)free_skb释放已发送完成的缓冲区
   b)执行qdisc_run->qdisc_restart->hard_start_xmit发送报文，会不停的发送，直到驱动通过netif_stop_queue停止出口队列

注意：出口队列停止后应该能在一定的时间之内恢复，看门狗定时器用于监控设备是否在给定时间内恢复，如果不能恢复会调用驱动程序提供的tx_timeout函数。
      该函数会复位网卡，然后以netif_wake_queue重启接口队列。
```
