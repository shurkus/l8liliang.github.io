---
layout: article
tags: OVN
title: OVN - Interconnection
mathjax: true
key: Linux
---


## topo
```

      +-----------------------------------------------------------------------------------------------------------------------+
      |ts1                                                                                                                    |
      |                                                                                                                       |
      |                                                                                                                       |
      |                                                                                                                       |
      |                                                                                                                       |
      |                  ts1-lr1                                                                         ts1-lr2              |
      +--------------------+--------------------------------------------------------------------------------+-----------------+
                           |                                                                                |
                           |                                                                                |
                           |                                                                                |
                           |                                                                                |
                           |                                                                                |
+--------------------------o-----------------------------------+     +--------------------------------------o----------------------------+
| ovn az1                  |                                   |     | ovn az2                              |                            | 
|                          |                                   |     |                                      |                            |
|     +--------------------+----------------+                  |     |                   +------------------+---------------------+      |
|     |lr1             lr1-ts1              |                  |     |                   |lr2            lr2-ts1                  |      |
|     |                                     |                  |     |                   |                                        |      |
|     |                                     |                  |     |                   |                                        |      |
|     |                                     |                  |     |                   |                                        |      |
|     |                                     |                  |     |                   |                                        |      |
|     |                lr1-ls1              |                  |     |                   |         lr2-ls2                        |      |
|     +-------------------+-----------------+                  |     |                   +-----------+----------------------------+      |
|                         |                                    |     |                               |                                   |
|                         |                                    |     |                               |                                   |
|                         |                                    |     |                               |                                   |
|                         |                                    |     |                               |                                   |
|                         |                                    |     |                               |                                   |
|                         |                                    |     |                               |                                   |
|                         |                                    |     |                               |                                   |
|                         |                                    |     |                               |                                   |
|     +-------------------+---------------------------------+  |     |      +------------------------+----------------------------+      |
|     |ls1             ls1-lr1                              |  |     |      |ls2                  ls2-lr2                         |      |
|     |                                                     |  |     |      |                                                     |      |
|     |h0_v0_vnet1,  h0_v1_vnet1,  h1_v0_vnet1,  h1_v1_vnet1|  |     |      |h0_v0_vnet2,  h0_v1_vnet2,  h1_v0_vnet2,  h1_v1_vnet2|      |
|     +-----+-------------+-------------+-------------+-----+  |     |      +-----+-------------+-------------+-------------+-----+      |
|           |             |                                    |     |                                        |             |            |
|           |             |                                    |     |                                        |             |            |
|           |             |                                    |     |                                        |             |            |
|           |             |                                    |     |                                        |             |            |
|           |             |                                    |     |                                        |             |            |
|           |             |                                    |     |                 +----------------------+             |            |
|           |             |                                    |     |                 |                                    |            |
|           |             |                                    |     |                 |                            +-------+            |
|           |             |                                    |     |                 |                            |                    |
|           |             |                                    |     |                 |                            |                    |
|           |             |                                    |     |                 |                            |                    |
|           |             +----------+                         |     |                 |                            |                    |
|           |                        |                         |     |                 |                            |                    |
|           |                        |                         |     |                 |                            |                    |
|     +-----o------------------------o----------------------+  |     |      +----------o----------------------------o--------------+     |
|     |     |                        |                      |  |     |      |          |                            |              |     |
|     |  +--+---------+            +-+----------+           |  |     |      |  +-------+----+             +-+-------+--+           |     |
|     |  |v0          |            |v1          |           |  |     |      |  |v0          |             |v1          |           |     |
|     |  +------------+            +------------+           |  |     |      |  +------------+             +------------+           |     |
|     |                                                     |  |     |      |                                                      |     |
|     |h0(server+computing+ic_ovsdb)                        |  |     |      |h1(server+computing)                                  |     |
|     +-----------------------------------------------------+  |     |      +------------------------------------------------------+     |
|                                                              |     |                                                                   |
+--------------------------------------------------------------+     +-------------------------------------------------------------------+

```


## summary
ovn Interconnection可以把两个ovn逻辑网络连接起来（中间走路由）。
它是通过把两个ovn的gateway router通过transit switch连接起来实现的。

Interconnection需要额外的全局ovsdb server，包括ovn-ic-sb,ovn-ic-nb，用来存储全局（所有子ovn都用到）信息。  
这个global ovsdb server可以放到单独的服务器上，也可放放到某个ovn central node上，但是要保证所有的ovn central node都能访问到。  
每个ovn的cnetral node需要开启ovn ic服务，并且通过ovn-ic-controller访问global ovsdb。  

Interconnection相当于又包了一层ovn（global ovsdb，ovn-ic-controller...）  
在本次实验中，
h0作为ovn az1的central node和Interconnection的global ovsdb server的驻留点，ip是177.1.1.1/24。  
h1作为ovn az2点central node，ip是177.1.1.2/24。  
az=availability zone  


## 使用脚本安装虚拟机
在h0和h1上面各安装两个虚拟机v0和v1。

```
rhel_version=rhel$(rpm -E %rhel)
  
# libvirt && kvm
yum -y install virt-install
yum -y install libvirt
yum install -y python3-lxml.x86_64
rpm -qa | grep qemu-kvm >/dev/null || yum -y install qemu-kvm
if (($rhel_version < 7)); then
        service libvirtd restart
else
        systemctl restart libvirtd
        systemctl start virtlogd.socket
fi

# work around for failure of virt-install
chmod 666 /dev/kvm


# define default vnet
virsh net-define /usr/share/libvirt/networks/default.xml
virsh net-start default
virsh net-autostart default

# define vm name and mac
vm_name=v1
mac4vm=a4:a4:a4:a4:a4:a1

# download image
wget http://netqe-bj.usersys.redhat.com/share/vms/rhel8.4.qcow2 -O /var/lib/libvirt/images/$vm_name.qcow2

# install vm
virt-install \
        --name $vm_name \
        --vcpus=2 \
        --ram=2048 \
        --disk path=/var/lib/libvirt/images/$vm_name.qcow2,device=disk,bus=virtio,format=qcow2 \
        --network bridge=virbr0,model=virtio,mac=$mac4vm \
        --boot hd \
        --accelerate \
        --graphics vnc,listen=0.0.0.0 \
        --force \
        --os-type=linux \
        --noautoconsol
```



## start ovn on hv0 and attach vf to vm

```
systemctl start openvswitch
systemctl start ovn-northd
ovn-sbctl set-connection ptcp:6642
ovn-nbctl set-connection ptcp:6641

ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:177.1.1.1:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=177.1.1.1
systemctl restart ovn-controller


mac_v0_vnet1=04:ac:10:ff:01:94
mac_v1_vnet1=04:ac:10:ff:01:96

cat <<-EOF > v0-vnet1.xml 
<interface type='bridge'>
        <target dev='h0_v0_vnet1'/>
        <mac address='${mac_v0_vnet1}'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>
EOF

cat <<-EOF > v1-vnet1.xml 
<interface type='bridge'>
        <target dev='h0_v1_vnet1'/>
        <mac address='${mac_v1_vnet1}'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>
EOF

virsh attach-device v0 v0-vnet1.xml
virsh attach-device v1 v1-vnet1.xml

ovs-vsctl set interface h0_v0_vnet1 external-ids:iface-id=h0_v0_vnet1
ovs-vsctl set interface h0_v1_vnet1 external-ids:iface-id=h0_v1_vnet1

```


## create az1 ovn topo on hv0
```
mac_h0_v0_vnet1=04:ac:10:ff:01:94
mac_h0_v0_vnet2=04:ac:10:ff:01:95
mac_h0_v1_vnet1=04:ac:10:ff:01:96
mac_h0_v1_vnet2=04:ac:10:ff:01:97
mac_h1_v0_vnet1=02:ac:10:ff:01:94
mac_h1_v0_vnet2=02:ac:10:ff:01:95
mac_h1_v1_vnet1=02:ac:10:ff:01:96
mac_h1_v1_vnet2=02:ac:10:ff:01:97

# add logical switch
ovn-nbctl ls-add ls1 -- add Logical_Switch ls1 other_config subnet=172.16.1.0/24
#ovn-nbctl ls-add ls2 -- add Logical_Switch ls2 other_config subnet=172.16.2.0/24

# setup ls ipv6_prefix
ovn-nbctl set Logical-switch ls1 other_config:ipv6_prefix=2001:db8:1::0
#ovn-nbctl set Logical-switch ls2 other_config:ipv6_prefix=2001:db8:2::0

# create dhcp_options
dhcp_options1=$(ovn-nbctl create DHCP_Options cidr=172.16.1.0/24 \
        options="\"server_id\"=\"172.16.1.254\" \"server_mac\"=\"00:00:00:00:01:00\" \
        \"lease_time\"=\"$((36000 + RANDOM % 3600))\" \"router\"=\"172.16.1.254\" \"dns_server\"=\"172.16.1.254\"")
#dhcp_options2=$(ovn-nbctl create DHCP_Options cidr=172.16.2.0/24 \
#        options="\"server_id\"=\"172.16.2.254\" \"server_mac\"=\"00:00:00:00:02:00\" \
#        \"lease_time\"=\"$((36000 + RANDOM % 3600))\" \"router\"=\"172.16.2.254\" \"dns_server\"=\"172.16.2.254\"")

dhcpv6_options1=$(ovn-nbctl create DHCP_Options cidr="2001\:db8\:1\:\:0/64" \
                                options="\"server_id\"=\"00:00:00:00:01:00\" \"dns_server\"=\"2001:db8:1::254\"")
#dhcpv6_options2=$(ovn-nbctl create DHCP_Options cidr="2001\:db8\:2\:\:0/64" \
#                                options="\"server_id\"=\"00:00:00:00:02:00\" \"dns_server\"=\"2001:db8:2::254\"")

# create logical switch port and setup dhcp_option
lsp_name=h0_v0_vnet1
mac=$mac_h0_v0_vnet1
ovn-nbctl lsp-add ls1 $lsp_name
ovn-nbctl lsp-set-addresses $lsp_name "$mac dynamic"
ovn-nbctl lsp-set-dhcpv4-options $lsp_name ${dhcp_options1}
ovn-nbctl add Logical_Switch_Port $lsp_name dhcpv6_options ${dhcpv6_options1}

lsp_name=h0_v1_vnet1
mac=$mac_h0_v1_vnet1
ovn-nbctl lsp-add ls1 $lsp_name
ovn-nbctl lsp-set-addresses $lsp_name "$mac dynamic"
ovn-nbctl lsp-set-dhcpv4-options $lsp_name ${dhcp_options1}
ovn-nbctl add Logical_Switch_Port $lsp_name dhcpv6_options ${dhcpv6_options1}

# create logical router lr1
ovn-nbctl lr-add lr1
ovn-nbctl lrp-add lr1 lr1-ls1 00:00:00:00:01:00 172.16.1.254/24

# connect lr1 and ls1
ovn-nbctl lsp-add ls1 ls1-lr1 
ovn-nbctl lsp-set-type ls1-lr1 router
ovn-nbctl lsp-set-addresses ls1-lr1 00:00:00:00:01:00 
ovn-nbctl lsp-set-options ls1-lr1 router-port=lr1-ls1


```

## start ovn on hv1 and attach vf to vm
```
systemctl start openvswitch
systemctl start ovn-northd
ovn-sbctl set-connection ptcp:6642
ovn-nbctl set-connection ptcp:6641

ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:177.1.1.2:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=177.1.1.2
systemctl restart ovn-controller

mac_v0_vnet1=02:ac:10:ff:01:94
mac_v0_vnet2=02:ac:10:ff:01:95
mac_v1_vnet1=02:ac:10:ff:01:96
mac_v1_vnet2=02:ac:10:ff:01:97

cat <<-EOF > v0-vnet2.xml
<interface type='bridge'>
        <target dev='h1_v0_vnet2'/>
        <mac address='${mac_v0_vnet2}'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>
EOF

cat <<-EOF > v1-vnet2.xml
<interface type='bridge'>
        <target dev='h1_v1_vnet2'/>
        <mac address='${mac_v1_vnet2}'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>
EOF

virsh attach-device v0 v0-vnet2.xml
virsh attach-device v1 v1-vnet2.xml

ovs-vsctl set interface h1_v0_vnet2 external-ids:iface-id=h1_v0_vnet2
ovs-vsctl set interface h1_v1_vnet2 external-ids:iface-id=h1_v1_vnet2
```

## create az2 ovn topo on hv1
```
mac_h0_v0_vnet1=04:ac:10:ff:01:94
mac_h0_v0_vnet2=04:ac:10:ff:01:95
mac_h0_v1_vnet1=04:ac:10:ff:01:96
mac_h0_v1_vnet2=04:ac:10:ff:01:97
mac_h1_v0_vnet1=02:ac:10:ff:01:94
mac_h1_v0_vnet2=02:ac:10:ff:01:95
mac_h1_v1_vnet1=02:ac:10:ff:01:96
mac_h1_v1_vnet2=02:ac:10:ff:01:97

# add logical switch
ovn-nbctl ls-add ls2 -- add Logical_Switch ls2 other_config subnet=172.16.2.0/24

# setup ls ipv6_prefix
ovn-nbctl set Logical-switch ls2 other_config:ipv6_prefix=2001:db8:2::0

# create dhcp_options
dhcp_options1=$(ovn-nbctl create DHCP_Options cidr=172.16.2.0/24 \
        options="\"server_id\"=\"172.16.2.254\" \"server_mac\"=\"00:00:00:00:02:00\" \
        \"lease_time\"=\"$((36000 + RANDOM % 3600))\" \"router\"=\"172.16.2.254\" \"dns_server\"=\"172.16.2.254\"")

dhcpv6_options1=$(ovn-nbctl create DHCP_Options cidr="2001\:db8\:2\:\:0/64" \
                                options="\"server_id\"=\"00:00:00:00:02:00\" \"dns_server\"=\"2001:db8:2::254\"")

# create logical switch port and setup dhcp_option
lsp_name=h1_v0_vnet2
mac=$mac_h1_v0_vnet2
ovn-nbctl lsp-add ls2 $lsp_name
ovn-nbctl lsp-set-addresses $lsp_name "$mac dynamic"
ovn-nbctl lsp-set-dhcpv4-options $lsp_name ${dhcp_options1}
ovn-nbctl add Logical_Switch_Port $lsp_name dhcpv6_options ${dhcpv6_options1}

lsp_name=h1_v1_vnet2
mac=$mac_h1_v1_vnet2
ovn-nbctl lsp-add ls2 $lsp_name
ovn-nbctl lsp-set-addresses $lsp_name "$mac dynamic"
ovn-nbctl lsp-set-dhcpv4-options $lsp_name ${dhcp_options1}
ovn-nbctl add Logical_Switch_Port $lsp_name dhcpv6_options ${dhcpv6_options1}

# create logical router lr2
ovn-nbctl lr-add lr2
ovn-nbctl lrp-add lr2 lr2-ls2 00:00:00:00:02:00 172.16.2.254/24

# connect lr2 and ls2
ovn-nbctl lsp-add ls2 ls2-lr2 
ovn-nbctl lsp-set-type ls2-lr2 router
ovn-nbctl lsp-set-addresses ls2-lr2 00:00:00:00:02:00 
ovn-nbctl lsp-set-options ls2-lr2 router-port=lr2-ls2

```


## setup ic on hv0
```
# start global ovsdb server
alias ovn-ctl="/usr/share/ovn/scripts/ovn-ctl"
ovn-ctl --db-ic-nb-create-insecure-remote=yes \
          --db-ic-sb-create-insecure-remote=yes start_ic_ovsdb

# add transit switch
ovn-ic-nbctl ts-add ts1

# set az name
ovn-nbctl set NB_Global . name=az1

# connecto to global ovsdb
ovn-ctl --ovn-ic-nb-db=tcp:177.1.1.1:6645 --ovn-ic-sb-db=tcp:177.1.1.1:6646 --ovn-northd-nb-db=tcp:177.1.1.1:6641 --ovn-northd-sb-db=tcp:177.1.1.1:6642 start_ic

# enable me as gateway chassis
ovs-vsctl set open_vswitch . external_ids:ovn-is-interconn=true

# connect gateway router to transit switch
ovn-nbctl lrp-add lr1 lr1-ts1 aa:aa:aa:aa:aa:01 169.254.100.1/24
ovn-nbctl lsp-add ts1 ts1-lr1 -- \
        lsp-set-addresses ts1-lr1 router -- \
        lsp-set-type ts1-lr1 router -- \
        lsp-set-options ts1-lr1 router-port=lr1-ts1

# setup distributed gateway port
ovn-nbctl lrp-set-gateway-chassis lr1-ts1 dfe51fb0-0c76-437c-ad95-709b98998b4f
```


## setup ic on hv1
```
alias ovn-ctl="/usr/share/ovn/scripts/ovn-ctl"
#ovn-ctl --db-ic-nb-create-insecure-remote=yes \
#          --db-ic-sb-create-insecure-remote=yes start_ic_ovsdb

# set az name
ovn-nbctl set NB_Global . name=az2

# connecto to global ovsdb
ovn-ctl --ovn-ic-nb-db=tcp:177.1.1.1:6645 --ovn-ic-sb-db=tcp:177.1.1.1:6646 --ovn-northd-nb-db=tcp:177.1.1.2:6641 --ovn-northd-sb-db=tcp:177.1.1.2:6642 start_ic

# enable me as gateway chassis
ovs-vsctl set open_vswitch . external_ids:ovn-is-interconn=true

# connect gateway router to transit switch
ovn-nbctl lrp-add lr2 lr2-ts1 aa:aa:aa:aa:aa:02 169.254.100.2/24
ovn-nbctl lsp-add ts1 ts1-lr2 -- \
        lsp-set-addresses ts1-lr2 router -- \
        lsp-set-type ts1-lr2 router -- \
        lsp-set-options ts1-lr2 router-port=lr2-ts1


# setup distributed gateway port
ovn-nbctl lrp-set-gateway-chassis lr2-ts1 68b0888a-06d3-4cb7-8da3-f647f69c83af
```


## setup route on both ovn central nodes
```
# static route
ovn-nbctl lr-route-add lr1 10.0.2.0/24 169.254.100.2

# dynamic learning
ovn-nbctl set NB_Global . options:ic-route-adv=true \
                            options:ic-route-learn=true

```

## ping crossing ovn
```
[root@localhost ~]# ping 172.16.2.2
PING 172.16.2.2 (172.16.2.2) 56(84) bytes of data.
64 bytes from 172.16.2.2: icmp_seq=1 ttl=62 time=2.46 ms
```


## cleanup
```
systemctl stop ovn-controller &>/dev/null
systemctl stop ovn-northd &>/dev/null
systemctl stop openvswitch &>/dev/null
sleep 5
rm -rf /etc/openvswitch/*.db
rm -rf /etc/openvswitch/*.pem
rm -rf /var/lib/openvswitch/*
rm -rf /var/lib/ovn/*
rm -rf /etc/ovn/*.db
rm -rf /etc/ovn/*.pem
# clean up log
rm -rf /var/log/ovn/*
rm -rf /var/log/openvswitch/*
```

[OVN Interconnection](https://docs.ovn.org/en/latest/tutorials/ovn-interconnection.html#)
{:.info}
