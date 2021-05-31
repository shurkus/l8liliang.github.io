---
layout: article
tags: OVN
title: OVN实验一：使用OVN配置交换机
mathjax: true
key: Linux
---

[参考](https://blog.csdn.net/zhengmx100/category_9268063.html)
{:.info}

## install ovn 
```
export RPM_OVS=http://download-node-02.eng.bos.redhat.com/brewroot/packages/openvswitch2.13/2.13.0/112.el8fdp/x86_64/openvswitch2.13-2.13.0-112.el8fdp.x86_64.rpm
export RPM_OVN_COMMON=http://download-node-02.eng.bos.redhat.com/brewroot/packages/ovn2.13/20.12.0/121.el8fdp/x86_64/ovn2.13-20.12.0-121.el8fdp.x86_64.rpm
export RPM_SELINUX=http://download-node-02.eng.bos.redhat.com/brewroot/packages/openvswitch-selinux-extra-policy/1.0/28.el8fdp/noarch/openvswitch-selinux-extra-policy-1.0-28.el8fdp.noarch.rpm
export RPM_PYTHON_OVS=
export RPM_OVN_CENTRAL=
export RPM_OVN_HOST=

source /mnt/tests/kernel/networkin/common/include.sh
source /mnt/tests/kernel/networking/openvswitch/ovn/common/include.sh 
ovn_install
```

## setup ovn master
```
# 启动服务
systemctl start openvswitch
systemctl start ovn-northd
ovn-sbctl set-connection ptcp:6642
ovn-nbctl set-connection ptcp:6641
systemctl restart ovn-controller

# 创建logical switch
ovn-nbctl ls-add ls1

# 创建 logical port
ovn-nbctl lsp-add ls1 ls1-vm1
ovn-nbctl lsp-set-addresses ls1-vm1 02:ac:10:ff:00:11
ovn-nbctl lsp-set-port-security ls1-vm1 02:ac:10:ff:00:11

# 创建 logical port
ovn-nbctl lsp-add ls1 ls1-vm2
ovn-nbctl lsp-set-addresses ls1-vm2 02:ac:10:ff:00:22
ovn-nbctl lsp-set-port-security ls1-vm2 02:ac:10:ff:00:22

ovn-nbctl show

ovn-sbctl show

```

## setup ovn-controller(slave)
```
######## hp-dl380pg8-15.rhts.eng.pek2.redhat.com:
systemctl start openvswitch
ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:177.1.1.1:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=177.1.1.2
systemctl start ovn-controller

<interface type='bridge'>
        <target dev='ovn-vnet0'/>
        <mac address='02:ac:10:ff:00:11'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>

virsh attach-device v0 vnet0.xml 

ovs-vsctl set interface ovn-vnet0 external-ids:iface-id=ls1-vm1


######## dell-per730-42.rhts.eng.pek2.redhat.com:
systemctl start openvswitch
ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:177.1.1.1:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=177.1.1.3
systemctl start ovn-controller

<interface type='bridge'>
        <target dev='ovn-vnet1'/>
        <mac address='02:ac:10:ff:00:22'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>

virsh attach-device v0 vnet1.xml 

ovs-vsctl set interface ovn-vnet1 external-ids:iface-id=ls1-vm2


iptables -F
setenforce 0
systemctl --now stop firewalld

```

## install_vm.sh
使用该脚本在ovn slave上创建虚拟机
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
vm_name=v0
mac4vm=a4:a4:a4:a4:a4:a0

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

## ping
```
[root@hp-dl380pg8-15 ~]# virsh console v0

[root@localhost ~]# ping 178.1.1.3
PING 178.1.1.3 (178.1.1.3) 56(84) bytes of data.
64 bytes from 178.1.1.3: icmp_seq=1 ttl=64 time=2.12 ms
64 bytes from 178.1.1.3: icmp_seq=2 ttl=64 time=0.326 ms

--- 178.1.1.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.326/1.223/2.120/0.897 ms
```

## Northbound DB
Northbound DB 是 OVN 和 CMS 之间的接口，Northbound DB 里面的几乎所有的内容都是由 CMS 产生的，ovn-northd 监听这个数据库的内容变化，然后翻译，保存到 Southbound DB 里面。

Northbound DB 里面主要有如下几张表：

* Logical_Switch：每一行代表一个逻辑交换机，逻辑交换机有两种，一种是 overlay logical switches，对应于 neutron network，每创建一个 neutron network，networking-ovn 会在这张表里增加一行；
另一种是 bridged logical switch，连接物理网络和逻辑网络，被 VTEP gateway 使用。Logical_Switch 里面保存了它包含的 logical port（指向 Logical_Port table）和应用在它上面的 ACL（指向 ACL table）。

* Logical_Port：每一行代表一个逻辑端口，每创建一个 neutron port，networking-ovn 会在这张表里增加一行，每行保存的信息有端口的类型，
比如 patch port，localnet port，端口的 IP 和 MAC 地址，端口的状态 UP/Down。

* ACL：每一行代表一个应用到逻辑交换机上的 ACL 规则，如果逻辑交换机上面的所有端口都没有配置 security group，那么这个逻辑交换机上不应用 ACL。每条 ACL 规则包含匹配的内容，方向，还有动作。

* Logical_Router：每一行代表一个逻辑路由器，每创建一个 neutron router，networking-ovn 会在这张表里增加一行，每行保存了它包含的逻辑的路由器端口。

* Logical_Router_Port：每一行代表一个逻辑路由器端口，每创建一个 router interface，networking-ovn 会在这张表里加一行，它主要保存了路由器端口的 IP 和 MAC。

```
[root@hp-dl388g8-22 ~]# ovn-nbctl list NB_Global
_uuid               : 8482ac67-d73e-48e6-a222-32cf7b54ee10
connections         : [538ca6cf-e9e8-49c6-8b01-8d377e229363]
external_ids        : {}
hv_cfg              : 0
hv_cfg_timestamp    : 0
ipsec               : false
name                : ""
nb_cfg              : 0
nb_cfg_timestamp    : 0
options             : {mac_prefix="de:aa:2a", max_tunid="16711680", northd_internal_version="20.12.0-20.16.1-56.0", svc_monitor_mac="82:51:e0:8b:2a:7c"}
sb_cfg              : 0
sb_cfg_timestamp    : 0
ssl                 : []

[root@hp-dl388g8-22 ~]# ovn-nbctl list Logical_Switch
_uuid               : aa65a602-642f-4a90-a0ef-1b1c2949d409
acls                : []
dns_records         : []
external_ids        : {}
forwarding_groups   : []
load_balancer       : []
name                : ls1
other_config        : {}
ports               : [237f8ad8-e3a5-42ca-b4c6-e54278b28bcb, f89aea67-8ad4-4b3e-bb36-95ef64db2aec]
qos_rules           : []

[root@hp-dl388g8-22 ~]# ovn-nbctl list Logical_Switch_Port
_uuid               : 237f8ad8-e3a5-42ca-b4c6-e54278b28bcb
addresses           : ["02:ac:10:ff:00:11"]
dhcpv4_options      : []
dhcpv6_options      : []
dynamic_addresses   : []
enabled             : []
external_ids        : {}
ha_chassis_group    : []
name                : ls1-vm1
options             : {}
parent_name         : []
port_security       : ["02:ac:10:ff:00:11"]
tag                 : []
tag_request         : []
type                : ""
up                  : true

_uuid               : f89aea67-8ad4-4b3e-bb36-95ef64db2aec
addresses           : ["02:ac:10:ff:00:22"]
dhcpv4_options      : []
dhcpv6_options      : []
dynamic_addresses   : []
enabled             : []
external_ids        : {}
ha_chassis_group    : []
name                : ls1-vm2
options             : {}
parent_name         : []
port_security       : ["02:ac:10:ff:00:22"]
tag                 : []
tag_request         : []
type                : ""
up                  : true

[root@hp-dl388g8-22 ~]# ovn-nbctl list ACL
```


## Southbound DB
Southbound DB 里面有如下几张表：

* Chassis：chassis这个概念, Chassis 是 OVN 新增的概念，OVS 里面没有这个概念。 
chassis表的每一行表示一个 HV 或者 VTEP 网关，由 ovn-controller/ovn-controller-vtep 填写，
包含 chassis 的名字和 chassis 支持的封装的配置（指向表 Encap），
如果 chassis 是 VTEP 网关，VTEP 网关上和 OVN 关联的逻辑交换机也保存在这张表里。

* Encap：保存着 tunnel 的类型和 tunnel endpoint IP 地址。

* Logical_Flow：每一行表示一个逻辑的流表，这张表是 ovn-northd 根据 Nourthbound DB 里面二三层拓扑信息和 ACL 信息转换而来的，
ovn-controller 把这个表里面的流表转换成 OVS 流表，配到 HV 上的 OVS table。流表主要包含匹配的规则，匹配的方向，优先级，table ID 和执行的动作。

* Multicast_Group：每一行代表一个组播组，组播报文和广播报文的转发由这张表决定，它保存了组播组所属的 datapath，组播组包含的端口，还有代表 logical egress port 的 tunnel_key。

* Datapath_Binding：每一行代表一个 datapath 和物理网络的绑定关系，每个 logical switch 和 logical router 对应一行。
它主要保存了 OVN 给 datapath 分配的代表 logical datapath identifier 的 tunnel_key。

* Port_Binding：这张表主要用来确定 logical port 处在哪个 chassis 上面。每一行包含的内容主要有 logical port 的 MAC 和 IP 地址，端口类型，
端口属于哪个 datapath binding，代表 logical input/output port identifier 的 tunnel_key, 以及端口处在哪个 chassis。
端口所处的 chassis 由 ovn-controller/ovn-controller 设置，其余的值由 ovn-northd 设置。

表 Chassis 和表 Encap 包含的是物理网络的数据，表 Logical_Flow 和表 Multicast_Group包含的是逻辑网络的数据，表 Datapath_Binding 和表 Port_Binding 包含的是逻辑网络和物理网络绑定关系的数据。

```
[root@hp-dl388g8-22 ~]# ovn-nbctl list Logical_Switch
_uuid               : aa65a602-642f-4a90-a0ef-1b1c2949d409
acls                : []
dns_records         : []
external_ids        : {}
forwarding_groups   : []
load_balancer       : []
name                : ls1
other_config        : {}
ports               : [237f8ad8-e3a5-42ca-b4c6-e54278b28bcb, f89aea67-8ad4-4b3e-bb36-95ef64db2aec]
qos_rules           : []
[root@hp-dl388g8-22 ~]# ovn-nbctl list NB_Global
_uuid               : 8482ac67-d73e-48e6-a222-32cf7b54ee10
connections         : [538ca6cf-e9e8-49c6-8b01-8d377e229363]
external_ids        : {}
hv_cfg              : 0
hv_cfg_timestamp    : 0
ipsec               : false
name                : ""
nb_cfg              : 0
nb_cfg_timestamp    : 0
options             : {mac_prefix="de:aa:2a", max_tunid="16711680", northd_internal_version="20.12.0-20.16.1-56.0", svc_monitor_mac="82:51:e0:8b:2a:7c"}
sb_cfg              : 0
sb_cfg_timestamp    : 0
ssl                 : []


[root@hp-dl388g8-22 ~]# ovn-nbctl list Logical_Switch_Port
_uuid               : 237f8ad8-e3a5-42ca-b4c6-e54278b28bcb
addresses           : ["02:ac:10:ff:00:11"]
dhcpv4_options      : []
dhcpv6_options      : []
dynamic_addresses   : []
enabled             : []
external_ids        : {}
ha_chassis_group    : []
name                : ls1-vm1
options             : {}
parent_name         : []
port_security       : ["02:ac:10:ff:00:11"]
tag                 : []
tag_request         : []
type                : ""
up                  : true

_uuid               : f89aea67-8ad4-4b3e-bb36-95ef64db2aec
addresses           : ["02:ac:10:ff:00:22"]
dhcpv4_options      : []
dhcpv6_options      : []
dynamic_addresses   : []
enabled             : []
external_ids        : {}
ha_chassis_group    : []
name                : ls1-vm2
options             : {}
parent_name         : []
port_security       : ["02:ac:10:ff:00:22"]
tag                 : []
tag_request         : []
type                : ""
up                  : true


[root@hp-dl388g8-22 ~]# ovn-nbctl list ACL


[root@hp-dl388g8-22 ~]# ovn-sbctl list SB_Global
_uuid               : fdb2824c-2f6e-4f05-a748-815bd9e8434a
connections         : [8f9ab75a-c266-4410-8e02-2ba7c07142fb]
external_ids        : {}
ipsec               : false
nb_cfg              : 0
options             : {mac_prefix="de:aa:2a", max_tunid="16711680", northd_internal_version="20.12.0-20.16.1-56.0", svc_monitor_mac="82:51:e0:8b:2a:7c"}
ssl                 : []


[root@hp-dl388g8-22 ~]# ovn-sbctl list Chassis
_uuid               : b671a1ec-154b-4161-a546-aec3b6d3b7b3
encaps              : [0a96ccff-d3ee-447b-8de1-a52138538f85]
external_ids        : {datapath-type="", iface-types="erspan,geneve,gre,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="", ovn-chassis-mac-mappings="", ovn-cms-options="", ovn-enable-lflow-cache="true", ovn-monitor-all="false", port-up-notif="true"}
hostname            : hp-dl380pg8-15.rhts.eng.pek2.redhat.com
name                : "330ddfec-e88e-4048-9be5-99e783e743c7"
nb_cfg              : 0
other_config        : {datapath-type="", iface-types="erspan,geneve,gre,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="", ovn-chassis-mac-mappings="", ovn-cms-options="", ovn-enable-lflow-cache="true", ovn-monitor-all="false", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : acb25bd6-4733-470f-b21b-6ad4df2c9b7c
encaps              : [e3ebe7a6-5168-4b11-8d66-ae9e47cbc2e5]
external_ids        : {datapath-type="", iface-types="erspan,geneve,gre,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="", ovn-chassis-mac-mappings="", ovn-cms-options="", ovn-enable-lflow-cache="true", ovn-monitor-all="false", port-up-notif="true"}
hostname            : dell-per730-42.rhts.eng.pek2.redhat.com
name                : "efa4d9b1-2a8e-45cb-af45-d628e075d836"
nb_cfg              : 0
other_config        : {datapath-type="", iface-types="erspan,geneve,gre,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="", ovn-chassis-mac-mappings="", ovn-cms-options="", ovn-enable-lflow-cache="true", ovn-monitor-all="false", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []


[root@hp-dl388g8-22 ~]# ovn-sbctl list Encap
_uuid               : e3ebe7a6-5168-4b11-8d66-ae9e47cbc2e5
chassis_name        : "efa4d9b1-2a8e-45cb-af45-d628e075d836"
ip                  : "177.1.1.3"
options             : {csum="true"}
type                : geneve

_uuid               : 0a96ccff-d3ee-447b-8de1-a52138538f85
chassis_name        : "330ddfec-e88e-4048-9be5-99e783e743c7"
ip                  : "177.1.1.2"
options             : {csum="true"}
type                : geneve


[root@hp-dl388g8-22 ~]# ovn-sbctl list Multicast_Group
_uuid               : b34ebbd3-f5f5-4d07-ab75-3a5eca51a8a6
datapath            : 15b2328c-ca1b-4b9d-9f05-c7f5f558025b
name                : _MC_flood
ports               : [1e9890f3-a054-44d7-bd8d-0ec845aad790, ee4c9a66-64c5-48d4-bf8b-5393839aa34b]
tunnel_key          : 32768

_uuid               : f87cf3e7-e3bf-4dcf-8bda-7309d99166d1
datapath            : 15b2328c-ca1b-4b9d-9f05-c7f5f558025b
name                : _MC_flood_l2
ports               : [1e9890f3-a054-44d7-bd8d-0ec845aad790, ee4c9a66-64c5-48d4-bf8b-5393839aa34b]
tunnel_key          : 32773


[root@hp-dl388g8-22 ~]# ovn-sbctl list Datapath_Binding
_uuid               : 15b2328c-ca1b-4b9d-9f05-c7f5f558025b
external_ids        : {logical-switch="aa65a602-642f-4a90-a0ef-1b1c2949d409", name=ls1}
load_balancers      : []
tunnel_key          : 1


[root@hp-dl388g8-22 ~]# ovn-sbctl list Port_Binding
_uuid               : ee4c9a66-64c5-48d4-bf8b-5393839aa34b
chassis             : acb25bd6-4733-470f-b21b-6ad4df2c9b7c
datapath            : 15b2328c-ca1b-4b9d-9f05-c7f5f558025b
encap               : []
external_ids        : {}
gateway_chassis     : []
ha_chassis_group    : []
logical_port        : ls1-vm2
mac                 : ["02:ac:10:ff:00:22"]
nat_addresses       : []
options             : {}
parent_port         : []
tag                 : []
tunnel_key          : 2
type                : ""
up                  : true
virtual_parent      : []

_uuid               : 1e9890f3-a054-44d7-bd8d-0ec845aad790
chassis             : b671a1ec-154b-4161-a546-aec3b6d3b7b3
datapath            : 15b2328c-ca1b-4b9d-9f05-c7f5f558025b
encap               : []
external_ids        : {}
gateway_chassis     : []
ha_chassis_group    : []
logical_port        : ls1-vm1
mac                 : ["02:ac:10:ff:00:11"]
nat_addresses       : []
options             : {}
parent_port         : []
tag                 : []
tunnel_key          : 1
type                : ""
up                  : true
virtual_parent      : []


[root@hp-dl388g8-22 ~]# ovn-sbctl list MAC_Binding


[root@hp-dl388g8-22 ~]# ovn-sbctl list Address_Set
_uuid               : 3827700c-eef1-4e75-a826-bcedc78e50d6
addresses           : ["82:51:e0:8b:2a:7c"]
name                : svc_monitor_mac


[root@hp-dl388g8-22 ~]# ovn-sbctl list Connection
_uuid               : 8f9ab75a-c266-4410-8e02-2ba7c07142fb
external_ids        : {}
inactivity_probe    : []
is_connected        : true
max_backoff         : []
other_config        : {}
read_only           : false
role                : ""
status              : {bound_port="6642", n_connections="2", sec_since_connect="0", sec_since_disconnect="0"}
target              : "ptcp:6642"


[root@hp-dl388g8-22 ~]# ovn-sbctl lflow-list
Datapath: "ls1" (15b2328c-ca1b-4b9d-9f05-c7f5f558025b)  Pipeline: ingress
  table=0 (ls_in_port_sec_l2  ), priority=100  , match=(eth.src[40]), action=(drop;)
  table=0 (ls_in_port_sec_l2  ), priority=100  , match=(vlan.present), action=(drop;)
  table=0 (ls_in_port_sec_l2  ), priority=50   , match=(inport == "ls1-vm1" && eth.src == {02:ac:10:ff:00:11}), action=(next;)
  table=0 (ls_in_port_sec_l2  ), priority=50   , match=(inport == "ls1-vm2" && eth.src == {02:ac:10:ff:00:22}), action=(next;)
  table=1 (ls_in_port_sec_ip  ), priority=0    , match=(1), action=(next;)
  table=2 (ls_in_port_sec_nd  ), priority=90   , match=(inport == "ls1-vm1" && eth.src == 02:ac:10:ff:00:11 && arp.sha == 02:ac:10:ff:00:11), action=(next;)
  table=2 (ls_in_port_sec_nd  ), priority=90   , match=(inport == "ls1-vm1" && eth.src == 02:ac:10:ff:00:11 && ip6 && nd && ((nd.sll == 00:00:00:00:00:00 || nd.sll == 02:ac:10:ff:00:11) || ((nd.tll == 00:00:00:00:00:00 || nd.tll == 02:ac:10:ff:00:11)))), action=(next;)
  table=2 (ls_in_port_sec_nd  ), priority=90   , match=(inport == "ls1-vm2" && eth.src == 02:ac:10:ff:00:22 && arp.sha == 02:ac:10:ff:00:22), action=(next;)
  table=2 (ls_in_port_sec_nd  ), priority=90   , match=(inport == "ls1-vm2" && eth.src == 02:ac:10:ff:00:22 && ip6 && nd && ((nd.sll == 00:00:00:00:00:00 || nd.sll == 02:ac:10:ff:00:22) || ((nd.tll == 00:00:00:00:00:00 || nd.tll == 02:ac:10:ff:00:22)))), action=(next;)
  table=2 (ls_in_port_sec_nd  ), priority=80   , match=(inport == "ls1-vm1" && (arp || nd)), action=(drop;)
  table=2 (ls_in_port_sec_nd  ), priority=80   , match=(inport == "ls1-vm2" && (arp || nd)), action=(drop;)
  table=2 (ls_in_port_sec_nd  ), priority=0    , match=(1), action=(next;)
  table=3 (ls_in_lookup_fdb   ), priority=0    , match=(1), action=(next;)
  table=4 (ls_in_put_fdb      ), priority=0    , match=(1), action=(next;)
  table=5 (ls_in_pre_acl      ), priority=110  , match=(eth.dst == $svc_monitor_mac), action=(next;)
  table=5 (ls_in_pre_acl      ), priority=0    , match=(1), action=(next;)
  table=6 (ls_in_pre_lb       ), priority=110  , match=(eth.dst == $svc_monitor_mac), action=(next;)
  table=6 (ls_in_pre_lb       ), priority=110  , match=(nd || nd_rs || nd_ra || mldv1 || mldv2), action=(next;)
  table=6 (ls_in_pre_lb       ), priority=0    , match=(1), action=(next;)
  table=7 (ls_in_pre_stateful ), priority=120  , match=(reg0[2] == 1 && ip4 && sctp), action=(reg1 = ip4.dst; reg2[0..15] = sctp.dst; ct_lb;)
  table=7 (ls_in_pre_stateful ), priority=120  , match=(reg0[2] == 1 && ip4 && tcp), action=(reg1 = ip4.dst; reg2[0..15] = tcp.dst; ct_lb;)
  table=7 (ls_in_pre_stateful ), priority=120  , match=(reg0[2] == 1 && ip4 && udp), action=(reg1 = ip4.dst; reg2[0..15] = udp.dst; ct_lb;)
  table=7 (ls_in_pre_stateful ), priority=120  , match=(reg0[2] == 1 && ip6 && sctp), action=(xxreg1 = ip6.dst; reg2[0..15] = sctp.dst; ct_lb;)
  table=7 (ls_in_pre_stateful ), priority=120  , match=(reg0[2] == 1 && ip6 && tcp), action=(xxreg1 = ip6.dst; reg2[0..15] = tcp.dst; ct_lb;)
  table=7 (ls_in_pre_stateful ), priority=120  , match=(reg0[2] == 1 && ip6 && udp), action=(xxreg1 = ip6.dst; reg2[0..15] = udp.dst; ct_lb;)
  table=7 (ls_in_pre_stateful ), priority=110  , match=(reg0[2] == 1), action=(ct_lb;)
  table=7 (ls_in_pre_stateful ), priority=100  , match=(reg0[0] == 1), action=(ct_next;)
  table=7 (ls_in_pre_stateful ), priority=0    , match=(1), action=(next;)
  table=8 (ls_in_acl_hint     ), priority=65535, match=(1), action=(next;)
  table=9 (ls_in_acl          ), priority=65535, match=(1), action=(next;)
  table=10(ls_in_qos_mark     ), priority=0    , match=(1), action=(next;)
  table=11(ls_in_qos_meter    ), priority=0    , match=(1), action=(next;)
  table=12(ls_in_stateful     ), priority=100  , match=(reg0[1] == 1), action=(ct_commit { ct_label.blocked = 0; }; next;)
  table=12(ls_in_stateful     ), priority=0    , match=(1), action=(next;)
  table=13(ls_in_pre_hairpin  ), priority=0    , match=(1), action=(next;)
  table=14(ls_in_nat_hairpin  ), priority=0    , match=(1), action=(next;)
  table=15(ls_in_hairpin      ), priority=0    , match=(1), action=(next;)
  table=16(ls_in_arp_rsp      ), priority=0    , match=(1), action=(next;)
  table=17(ls_in_dhcp_options ), priority=0    , match=(1), action=(next;)
  table=18(ls_in_dhcp_response), priority=0    , match=(1), action=(next;)
  table=19(ls_in_dns_lookup   ), priority=0    , match=(1), action=(next;)
  table=20(ls_in_dns_response ), priority=0    , match=(1), action=(next;)
  table=21(ls_in_external_port), priority=0    , match=(1), action=(next;)
  table=22(ls_in_l2_lkup      ), priority=110  , match=(eth.dst == $svc_monitor_mac), action=(handle_svc_check(inport);)
  table=22(ls_in_l2_lkup      ), priority=70   , match=(eth.mcast), action=(outport = "_MC_flood"; output;)
  table=22(ls_in_l2_lkup      ), priority=50   , match=(eth.dst == 02:ac:10:ff:00:11), action=(outport = "ls1-vm1"; output;)
  table=22(ls_in_l2_lkup      ), priority=50   , match=(eth.dst == 02:ac:10:ff:00:22), action=(outport = "ls1-vm2"; output;)
  table=22(ls_in_l2_lkup      ), priority=0    , match=(1), action=(outport = get_fdb(eth.dst); next;)
  table=23(ls_in_l2_unknown   ), priority=50   , match=(outport == "none"), action=(drop;)
  table=23(ls_in_l2_unknown   ), priority=0    , match=(1), action=(output;)
Datapath: "ls1" (15b2328c-ca1b-4b9d-9f05-c7f5f558025b)  Pipeline: egress
  table=0 (ls_out_pre_lb      ), priority=110  , match=(eth.src == $svc_monitor_mac), action=(next;)
  table=0 (ls_out_pre_lb      ), priority=110  , match=(nd || nd_rs || nd_ra || mldv1 || mldv2), action=(next;)
  table=0 (ls_out_pre_lb      ), priority=0    , match=(1), action=(next;)
  table=1 (ls_out_pre_acl     ), priority=110  , match=(eth.src == $svc_monitor_mac), action=(next;)
  table=1 (ls_out_pre_acl     ), priority=0    , match=(1), action=(next;)
  table=2 (ls_out_pre_stateful), priority=110  , match=(reg0[2] == 1), action=(ct_lb;)
  table=2 (ls_out_pre_stateful), priority=100  , match=(reg0[0] == 1), action=(ct_next;)
  table=2 (ls_out_pre_stateful), priority=0    , match=(1), action=(next;)
  table=3 (ls_out_acl_hint    ), priority=65535, match=(1), action=(next;)
  table=4 (ls_out_acl         ), priority=65535, match=(1), action=(next;)
  table=5 (ls_out_qos_mark    ), priority=0    , match=(1), action=(next;)
  table=6 (ls_out_qos_meter   ), priority=0    , match=(1), action=(next;)
  table=7 (ls_out_stateful    ), priority=100  , match=(reg0[1] == 1), action=(ct_commit { ct_label.blocked = 0; }; next;)
  table=7 (ls_out_stateful    ), priority=0    , match=(1), action=(next;)
  table=8 (ls_out_port_sec_ip ), priority=0    , match=(1), action=(next;)
  table=9 (ls_out_port_sec_l2 ), priority=100  , match=(eth.mcast), action=(output;)
  table=9 (ls_out_port_sec_l2 ), priority=50   , match=(outport == "ls1-vm1" && eth.dst == {02:ac:10:ff:00:11}), action=(output;)
  table=9 (ls_out_port_sec_l2 ), priority=50   , match=(outport == "ls1-vm2" && eth.dst == {02:ac:10:ff:00:22}), action=(output;)



[root@hp-dl388g8-22 ~]# ovn-sbctl list Logical_Flow   //只显示部分结果
_uuid               : 1c34ac80-d7df-4389-a92e-fb5ad5ab972f
actions             : "next;"
external_ids        : {source="ovn-northd.c:4580", stage-hint=f89aea67, stage-name=ls_in_port_sec_nd}
logical_datapath    : 15b2328c-ca1b-4b9d-9f05-c7f5f558025b
logical_dp_group    : []
match               : "inport == \"ls1-vm2\" && eth.src == 02:ac:10:ff:00:22 && ip6 && nd && ((nd.sll == 00:00:00:00:00:00 || nd.sll == 02:ac:10:ff:00:22) || ((nd.tll == 00:00:00:00:00:00 || nd.tll == 02:ac:10:ff:00:22)))"
pipeline            : ingress
priority            : 90
table_id            : 2
hash                : 0

_uuid               : a8312181-9aeb-4db2-bae0-9847ee87f6cd
actions             : "output;"
external_ids        : {source="ovn-northd.c:5129", stage-hint=f89aea67, stage-name=ls_out_port_sec_l2}
logical_datapath    : 15b2328c-ca1b-4b9d-9f05-c7f5f558025b
logical_dp_group    : []
match               : "outport == \"ls1-vm2\" && eth.dst == {02:ac:10:ff:00:22}"
pipeline            : egress
priority            : 50
table_id            : 9
hash                : 0
```
