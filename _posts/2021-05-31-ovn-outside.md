---
layout: article
tags: OVN
title: OVN实验三：OVN连接外部网络
mathjax: true
key: Linux
---

[参考](https://blog.csdn.net/zhengmx100/category_9268063.html)
{:.info}

## topo
```


              outside network
                 |
                 |
                 |
+----------------+------------------------------+
|              eth1                            |
|                                              |
|    sw outside                                |
|                                              |
|              outside-edge1                   |
+----------------------+-----------------------+
                       |
                       |
                       |
+----------------------------------------------+
|              edge1-outside                   |
|           02:0a:7f:00:01:29 10.127.0.129     |
|                                              |
|    Gateway router edge1                      |
|                                              |
|          02:ac:10:ff:00:01 172.16.255.1      |
|              edge1-transit                   |
+----------------------+-----------------------+
                       |
                       |
                       |
+----------------------+-----------------------+
|              transit-edge1                   |
|                                              |
|    sw transit                                |
|    172.16.255.0/30                           |
|                                              |
|              transit-tenant1                 |
+----------------------+-----------------------+
                       |
                       |
                       |
+----------------------+------------------------------------------------------+
|    router tenant1                                                           |
|                                                                             |
|                                                                             |
| 02:ac:10:ff:01:29                                       02:ac:10:ff:01:93   |
| 172.16.255.129                                          172.16.255.193      |
| tenant1-dmz                                             tenant1-inside      |
+----+----------------------------------------------------------+-------------+
     |                                                          |
     |                                                          |
     |                                                          |
+----+------------------------------+         +-----------------+-------------+
| dmz-tenant1                       |         |           inside-tenant1      |
|                                   |         |                               |
|             sw dmz                |         |           sw inside           |
|        172.16.255.128/26          |         |        172.16.255.192/26      |
|                                   |         |                               |
|  dmz-vm1                 dmz-vm2  |         | inside-vm1       inside-vm2   |
+-----+-----------------------+-----+         +-------+--------------+--------+
      |                       |                       |              |
      |                       |                       |              |
      |                       |                       |              |
+-----+------------+  +-------+-----------+  +--------+--------+   +-+----------------+
| 02:ac:10:ff:01:30|  | 02:ac:10:ff:01:31 |  |02:ac:10:ff:01:94|   | 02:ac:10:ff:01:95|
| 172.16.255.130   |  | 172.16.255.131    |  | 172.16.255.194  |   | 172.16.255.195   |
|      vm1         |  |      vm2          |  |     vm3         |   |     vm4          |
+-------------------  +-------------------+  +-----------------+   +------------------+
```

在实验二环境的基础上做下面的额外配置。
{:.info}

## 把ovn master也设置为ovn slave
```
ovs-vsctl set open . external-ids:ovn-remote=tcp:127.0.0.1:6642
ovs-vsctl set open . external-ids:ovn-encap-type=geneve
ovs-vsctl set open . external-ids:ovn-encap-ip=177.1.1.1

```

## 添加Gateway
```
gatway路由器将绑定到一个特定的 chassis。 为了完成这个绑定，我们需要找到 chassis ID。 
使用ovn-sbctl show命令，您应该能看到类似于此的输出:

[root@hp-dl388g8-22 ~]# ovn-sbctl show
Chassis "330ddfec-e88e-4048-9be5-99e783e743c7"
    hostname: hp-dl380pg8-15.rhts.eng.pek2.redhat.com
    Encap geneve
        ip: "177.1.1.2"
        options: {csum="true"}
    Port_Binding dmz-vm2
    Port_Binding dmz-vm1
Chassis "70586476-4abe-441b-bdec-fbea0971fa09"
    hostname: hp-dl388g8-22.rhts.eng.pek2.redhat.com
    Encap geneve
        ip: "177.1.1.1"
        options: {csum="true"}
    Port_Binding transit-edge1
    Port_Binding outside-edge1
    Port_Binding edge1-outside
    Port_Binding edge1-transit
Chassis "efa4d9b1-2a8e-45cb-af45-d628e075d836"
    hostname: dell-per730-42.rhts.eng.pek2.redhat.com
    Encap geneve
        ip: "177.1.1.3"
        options: {csum="true"}
    Port_Binding inside-vm3
    Port_Binding inside-vm4


# 创建 Gateway router edge1
ovn-nbctl create Logical_Router name=edge1 options:chassis={chassis_uuid}

#创建连接edge1和tenant1 routers的逻辑交换机transit
ovn-nbctl ls-add transit

# edge1 到transit switch
ovn-nbctl lrp-add edge1 edge1-transit 02:ac:10:ff:00:01 172.16.255.1/30
ovn-nbctl lsp-add transit transit-edge1
ovn-nbctl lsp-set-type transit-edge1 router
ovn-nbctl lsp-set-addresses transit-edge1 02:ac:10:ff:00:01
ovn-nbctl lsp-set-options transit-edge1 router-port=edge1-transit

# tenant1 到transit switch
ovn-nbctl lrp-add tenant1 tenant1-transit 02:ac:10:ff:00:02 172.16.255.2/30
ovn-nbctl lsp-add transit transit-tenant1
ovn-nbctl lsp-set-type transit-tenant1 router
ovn-nbctl lsp-set-addresses transit-tenant1 02:ac:10:ff:00:02
ovn-nbctl lsp-set-options transit-tenant1 router-port=tenant1-transit

#添加静态路由
ovn-nbctl lr-route-add edge1 "172.16.255.128/25" 172.16.255.2
ovn-nbctl lr-route-add tenant1 "0.0.0.0/0" 172.16.255.1

ovn-sbctl show

```

## vm1 此时可以ping通 edge1
```
[root@localhost ~]# ping 172.16.255.1
PING 172.16.255.1 (172.16.255.1) 56(84) bytes of data.
64 bytes from 172.16.255.1: icmp_seq=1 ttl=253 time=2.48 ms
```

## 连接到外部网络  
我们将使用eth1接口作为我们在edge1路由器和“外部”网络之间的连接点。
为了实现这一点，我们需要将OVN设置为通过专用OVS桥直接使用eth1接口。 
这种类型的连接在OVN中称为“localnet”。  
```
#在路由器 'edge1'创建新的端口
ovn-nbctl lrp-add edge1 edge1-outside 02:0a:7f:00:01:29 10.127.0.129/25

# 新建逻辑交换机，并将它连接到edge1
ovn-nbctl ls-add outside
ovn-nbctl lsp-add outside outside-edge1
ovn-nbctl lsp-set-type outside-edge1 router
ovn-nbctl lsp-set-addresses outside-edge1 02:0a:7f:00:01:29
ovn-nbctl lsp-set-options outside-edge1 router-port=edge1-outside

# 为 eth1新建OVS网桥
ovs-vsctl add-br br-eth1

# 为 eth1创建网桥映射： 把 "dataNet" 映射到 br-eth1
ovs-vsctl set Open_vSwitch . external-ids:ovn-bridge-mappings=dataNet:br-eth1

#在 'outside'交换机创建localnet端口。把网络名称设置为"dataNet"
ovn-nbctl lsp-add outside outside-localnet
ovn-nbctl lsp-set-addresses outside-localnet unknown
ovn-nbctl lsp-set-type outside-localnet localnet
ovn-nbctl lsp-set-options outside-localnet network_name=dataNet

# 将 eth1 连接到 br-eth1
ovs-vsctl add-port br-eth1 eth1

```

## vm1 此时可以ping通 edge1-outside
```
[root@localhost ~]# ping 10.127.0.129
PING 10.127.0.129 (10.127.0.129) 56(84) bytes of data.
64 bytes from 10.127.0.129: icmp_seq=1 ttl=253 time=1.97 ms
64 bytes from 10.127.0.129: icmp_seq=2 ttl=253 time=0.770 ms
```

## 但是vm1 不能ping通 外部网络的主机
```
因为外部主机没有到vm1 ip的路由，我们有2种方法来解决这个问题：
	在外部主机上添加静态路由
	在OVN网关路由器上设置NAT

使用第二种方法：
# create snat rule which will nat to the edge1-outside interface
ovn-nbctl -- --id=@nat create nat type="snat" logical_ip=172.16.255.128/25 external_ip=10.127.0.129 -- add logical_router edge1 nat @nat

简而言之，此命令在北向数据库的“nat”表中创建一个条目，再将同时产生的UUID存储在ovsdb的变量“@nat”中。
然后将存储在@nat中的UUID添加到北向数据库的“logical_router”表中的“edge1”条目的“nat”字段。

[root@localhost ~]# ping 10.127.0.133
PING 10.127.0.133 (10.127.0.133) 56(84) bytes of data.
64 bytes from 10.127.0.133: icmp_seq=1 ttl=62 time=2.70 ms
64 bytes from 10.127.0.133: icmp_seq=2 ttl=62 time=0.676 ms

# 下面是在外部主机抓到的包
04:22:54.135589 02:0a:7f:00:01:29 > 00:10:18:d3:0c:fe, ethertype IPv4 (0x0800), length 98: 10.127.0.129 > 10.127.0.133: ICMP echo request, id 5914, seq 7, length 64
04:22:54.135640 00:10:18:d3:0c:fe > 02:0a:7f:00:01:29, ethertype IPv4 (0x0800), length 98: 10.127.0.133 > 10.127.0.129: ICMP echo reply, id 5914, seq 7, length 64
```
