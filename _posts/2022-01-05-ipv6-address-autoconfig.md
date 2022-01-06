---
layout: article
tags: IPv6
title: IPv6 Address Auto Configuration
mathjax: true
key: Linux
---

[Reserved](https://www.kernel.org/doc/Documentation/networking/tuntap.txt)
{:.info} 

## IPv4 dhcp

```
1.dhcp discovery: 客户端以广播方式发出dhcp discovery消息，寻找本地子网内的dhcp服务器，
                    源地址是客户端最近一次获取的dhcp地址，或者0.0.0.0
2.dhcp offer: 子网内的一台或多台dhcp服务器回应dhcp offer消息，消息中包括一个建议的ip和掩码，
              还包括dhcp服务器的id，即服务器的ip地址
3.dhcp request: 客户端以广播的方式发出dhcp request，确认特定dhcp服务器的dhcp offer。
                消息中会包括dhcp服务器的id。使用广播的方式，是因为要向其他dhcp服务器通知自己的选择。
4.dhcp ack: dhcp服务器回复dhcp ack消息，确认客户端的ip。
```

## IPv6地址（公网单播）分配方法
```
1.无状态地址自动配置SLAAC
2.SLAAC和无状态dhcpv6服务器
  无状态dhcpv6只提供DNS、域名等信息，不提供IP。
3.有状态dhcpv6服务器
  有状态dhcpv6提供GUA、DNS、域名等信息。并且保存为每个主机分配地址的记录。

1.路由器会定期向所有ipv6设备ff02::1放送RA消息，cisco默认200秒间隔，RA消息里面携带前缀信息和标志位，
  如果A标志位打开，那么主机会根据前缀信息创建单播地址，即执行SLAAC
  如果O标志位打开，那么还会从无状态dhcpv6服务器获取其他信息，比如dns
  如果M标志位打开，那么需要从有状态dhcpv6服务器获取ip，dns等
  但是默认网关只能从RA消息的源ip获取

2.设备也可以主动发送RS消息查询，路由器会回复RA消息
```
