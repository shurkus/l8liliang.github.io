---
layout: article
tags: OVN
title: OVN实验二：使用OVN配置路由器
mathjax: true
key: Linux
---

[参考](https://blog.csdn.net/zhengmx100/category_9268063.html)
{:.info}

## topo
```
+-----------------------------------------------------------------------------+
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
+-----+------------+  +-------+-----------+  +--------+--------+   +-----+------------+
| 02:ac:10:ff:01:30|  | 02:ac:10:ff:01:31 |  |02:ac:10:ff:01:94|   | 02:ac:10:ff:01:95|
| 172.16.255.130   |  | 172.16.255.131    |  | 172.16.255.194  |   | 172.16.255.195   |
|      vm1         |  |      vm2          |  |     vm3         |   |     vm4          |
+-------------------  +-------------------+  +-----------------+   +------------------+
```

## 创建虚拟机
```
[root@hp-dl380pg8-15 ~]# cat vnet0.xml 
<interface type='bridge'>
        <target dev='ovn-vnet0'/>
        <mac address='02:ac:10:ff:01:30'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>

[root@hp-dl380pg8-15 ~]# cat vnet2.xml 
<interface type='bridge'>
        <target dev='ovn-vnet2'/>
        <mac address='02:ac:10:ff:01:31'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>

# virsh attach-device v0 vnet0.xml 
# virsh attach-device v1 vnet2.xml 
# ovs-vsctl set interface ovn-vnet0 external-ids:iface-id=dmz-vm1
# ovs-vsctl set interface ovn-vnet2 external-ids:iface-id=dmz-vm2


[root@dell-per730-42 ~]# cat vnet1.xml
<interface type='bridge'>
        <target dev='ovn-vnet1'/>
        <mac address='02:ac:10:ff:01:94'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>

[root@dell-per730-42 ~]# cat vnet3.xml
<interface type='bridge'>
        <target dev='ovn-vnet3'/>
        <mac address='02:ac:10:ff:01:95'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>

# virsh attach-device v0 vnet1.xml
# virsh attach-device v1 vnet3.xml
# ovs-vsctl set interface ovn-vnet1 external-ids:iface-id=inside-vm3
# ovs-vsctl set interface ovn-vnet3 external-ids:iface-id=inside-vm4
```

## 创建逻辑交换机和逻辑路由器
```
# 添加逻辑交换机
ovn-nbctl ls-add inside
ovn-nbctl ls-add dmz

# 添加逻辑路由器及其关联的路由器和交换机端口
# 添加路由器tenant1
ovn-nbctl lr-add tenant1

# 为路由器tenant1创建一个连接到dmz交换机的端口
ovn-nbctl lrp-add tenant1 tenant1-dmz 02:ac:10:ff:01:29 172.16.255.129/26

# 为dmz交换机创建用于连接到路由器tenant1的端口dmz-tenant1
ovn-nbctl lsp-add dmz dmz-tenant1
ovn-nbctl lsp-set-type dmz-tenant1 router
ovn-nbctl lsp-set-addresses dmz-tenant1 02:ac:10:ff:01:29
ovn-nbctl lsp-set-options dmz-tenant1 router-port=tenant1-dmz

#为路由器tenant1创建一个连接到inside交换机的端口
ovn-nbctl lrp-add tenant1 tenant1-inside 02:ac:10:ff:01:93 172.16.255.193/26

#为inside交换机创建用于连接到路由器tenant1的端口inside-tenant1
ovn-nbctl lsp-add inside inside-tenant1
ovn-nbctl lsp-set-type inside-tenant1 router
ovn-nbctl lsp-set-addresses inside-tenant1 02:ac:10:ff:01:93
ovn-nbctl lsp-set-options inside-tenant1 router-port=tenant1-inside

ovn-nbctl show
```

## 为虚拟机添加逻辑端口
```
ovn-nbctl lsp-add dmz dmz-vm1
ovn-nbctl lsp-set-addresses dmz-vm1 "02:ac:10:ff:01:30 172.16.255.130"
ovn-nbctl lsp-set-port-security dmz-vm1 "02:ac:10:ff:01:30 172.16.255.130"

ovn-nbctl lsp-add dmz dmz-vm2
ovn-nbctl lsp-set-addresses dmz-vm2 "02:ac:10:ff:01:31 172.16.255.131"
ovn-nbctl lsp-set-port-security dmz-vm2 "02:ac:10:ff:01:31 172.16.255.131"

ovn-nbctl lsp-add inside inside-vm3
ovn-nbctl lsp-set-addresses inside-vm3 "02:ac:10:ff:01:94 172.16.255.194"
ovn-nbctl lsp-set-port-security inside-vm3 "02:ac:10:ff:01:94 172.16.255.194"

ovn-nbctl lsp-add inside inside-vm4
ovn-nbctl lsp-set-addresses inside-vm4 "02:ac:10:ff:01:95 172.16.255.195"
ovn-nbctl lsp-set-port-security inside-vm4 "02:ac:10:ff:01:95 172.16.255.195"

ovn-nbctl show
```

## 为逻辑端口设置DHCP OPTION
```
dmzDhcp="$(ovn-nbctl create DHCP_Options cidr=172.16.255.128/26 \
options="\"server_id\"=\"172.16.255.129\" \"server_mac\"=\"02:ac:10:ff:01:29\" \
\"lease_time\"=\"3600\" \"router\"=\"172.16.255.129\"")"
echo $dmzDhcp

insideDhcp="$(ovn-nbctl create DHCP_Options cidr=172.16.255.192/26 \
options="\"server_id\"=\"172.16.255.193\" \"server_mac\"=\"02:ac:10:ff:01:93\" \
\"lease_time\"=\"3600\" \"router\"=\"172.16.255.193\"")"
echo $insideDhcp

ovn-nbctl dhcp-options-list

ovn-nbctl lsp-set-dhcpv4-options dmz-vm1 $dmzDhcp
ovn-nbctl lsp-get-dhcpv4-options dmz-vm1

ovn-nbctl lsp-set-dhcpv4-options dmz-vm2 $dmzDhcp
ovn-nbctl lsp-get-dhcpv4-options dmz-vm2

ovn-nbctl lsp-set-dhcpv4-options inside-vm3 $insideDhcp
ovn-nbctl lsp-get-dhcpv4-options inside-vm3

ovn-nbctl lsp-set-dhcpv4-options inside-vm4 $insideDhcp
ovn-nbctl lsp-get-dhcpv4-options inside-vm4
```

## 测试连通性
```
[root@hp-dl380pg8-15 ~]# virsh console v0
setlocale: No such file or directory
Connected to domain v0
Escape character is ^]

[root@localhost ~]# ping 172.16.255.129
PING 172.16.255.129 (172.16.255.129) 56(84) bytes of data.
64 bytes from 172.16.255.129: icmp_seq=1 ttl=254 time=1.27 ms
64 bytes from 172.16.255.129: icmp_seq=2 ttl=254 time=0.272 ms

[root@localhost ~]# ping 172.16.255.131
PING 172.16.255.131 (172.16.255.131) 56(84) bytes of data.
64 bytes from 172.16.255.131: icmp_seq=1 ttl=64 time=0.904 ms

[root@localhost ~]# ping 172.16.255.194
PING 172.16.255.194 (172.16.255.194) 56(84) bytes of data.
64 bytes from 172.16.255.194: icmp_seq=1 ttl=63 time=1.57 ms

[root@localhost ~]# ping 172.16.255.195
PING 172.16.255.195 (172.16.255.195) 56(84) bytes of data.
64 bytes from 172.16.255.195: icmp_seq=1 ttl=63 time=1.41 ms
```
