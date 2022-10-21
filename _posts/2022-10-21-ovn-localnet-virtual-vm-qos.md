---
layout: article
tags: OVN
title: OVN - localnet + virtualVM + qos
mathjax: true
key: Linux
---


## https://bugzilla.redhat.com/show_bug.cgi?id=2136716
```
port1=ens1f0np0
port2=ens1f1np1

ip link set $port1 up
ip addr add 177.1.1.1/16 dev $port1 &>/dev/null

systemctl start openvswitch
systemctl start ovn-northd
ovn-sbctl set-connection ptcp:6642
ovn-nbctl set-connection ptcp:6641

ovs-vsctl set Open_vSwitch . external-ids:system-id=hv1
ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:177.1.1.1:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=177.1.1.1
systemctl restart ovn-controller


# dhcp options
dhcp_102="$(ovn-nbctl create DHCP_Options cidr=172.16.102.0/24 \
        options="\"server_id\"=\"172.16.102.1\" \"server_mac\"=\"00:de:ad:ff:01:02\" \
        \"lease_time\"=\"3600\" \"router\"=\"172.16.102.1\"")" 

# r1
ovn-nbctl lr-add r1
ovn-nbctl lrp-add r1 r1_s2 00:de:ad:ff:01:02 172.16.102.1/24
ovn-nbctl lrp-add r1 r1_s3 00:de:ad:ff:01:03 172.16.103.1/24

# s2
ovn-nbctl ls-add s2

# s2 - r1
ovn-nbctl lsp-add s2 s2_r1
ovn-nbctl lsp-set-type s2_r1 router
#ovn-nbctl lsp-set-addresses s2_r1 00:de:ad:ff:01:02
ovn-nbctl lsp-set-addresses s2_r1 "00:de:ad:ff:01:02 172.16.102.1"
ovn-nbctl lsp-set-options s2_r1 router-port=r1_s2

# s2 - hv1_vm00_vnet1
ovn-nbctl lsp-add s2 hv1_vm00_vnet1
ovn-nbctl lsp-set-addresses hv1_vm00_vnet1 "00:de:ad:01:00:01 172.16.102.11"
ovn-nbctl lsp-set-dhcpv4-options hv1_vm00_vnet1 $dhcp_102

# s2 - hv1_vm01_vnet1
ovn-nbctl lsp-add s2 hv1_vm01_vnet1
ovn-nbctl lsp-set-addresses hv1_vm01_vnet1 "00:de:ad:01:01:01 172.16.102.12"
ovn-nbctl lsp-set-dhcpv4-options hv1_vm01_vnet1 $dhcp_102

# external network
ovs-vsctl add-br ext_net
ip link set ext_net up
ovs-vsctl set Open_vSwitch . external-ids:ovn-bridge-mappings=external_net:ext_net
ovs-vsctl add-port ext_net $port2
ip link set $port2 up

ovn-nbctl ls-add public
ovn-nbctl lsp-add public ln_p1
ovn-nbctl lsp-set-addresses ln_p1 unknown
ovn-nbctl lsp-set-type ln_p1 localnet
ovn-nbctl lsp-set-options ln_p1 network_name=external_net

ovn-nbctl lrp-add r1 r1_public 40:44:00:00:00:03 172.16.104.1/24
ovn-nbctl lsp-add public public_r1
ovn-nbctl lsp-set-type public_r1 router
ovn-nbctl lsp-set-addresses public_r1 router
ovn-nbctl lsp-set-options public_r1 router-port=r1_public nat-addresses=router
ovn-nbctl lrp-set-gateway-chassis r1_public hv1

# create virtual vm00
ovs-vsctl add-port br-int hv1_vm00_vnet1 -- set interface hv1_vm00_vnet1 type=internal
ip netns add hv1_vm00_vnet1
ip link set hv1_vm00_vnet1 netns hv1_vm00_vnet1
ip netns exec hv1_vm00_vnet1 ip link set lo up
ip netns exec hv1_vm00_vnet1 ip link set hv1_vm00_vnet1 up
ip netns exec hv1_vm00_vnet1 ip link set hv1_vm00_vnet1 address 00:de:ad:01:00:01
#pkill dhclient
#ip netns exec hv1_vm00_vnet1 dhclient -v hv1_vm00_vnet1
ip netns exec hv1_vm00_vnet1 ip addr add 172.16.102.11/24 dev hv1_vm00_vnet1
ip netns exec hv1_vm00_vnet1 ip route add default via 172.16.102.1 dev hv1_vm00_vnet1
ovs-vsctl set Interface hv1_vm00_vnet1 external_ids:iface-id=hv1_vm00_vnet1

# create virtual vm01
ovs-vsctl add-port br-int hv1_vm01_vnet1 -- set interface hv1_vm01_vnet1 type=internal
ip netns add hv1_vm01_vnet1
ip link set hv1_vm01_vnet1 netns hv1_vm01_vnet1
ip netns exec hv1_vm01_vnet1 ip link set lo up
ip netns exec hv1_vm01_vnet1 ip link set hv1_vm01_vnet1 up
ip netns exec hv1_vm01_vnet1 ip link set hv1_vm01_vnet1 address 00:de:ad:01:01:01
#pkill dhclient
#ip netns exec hv1_vm01_vnet1 dhclient -v hv1_vm01_vnet1
ip netns exec hv1_vm01_vnet1 ip addr add 172.16.102.12/24 dev hv1_vm01_vnet1
ip netns exec hv1_vm01_vnet1 ip route add default via 172.16.102.1 dev hv1_vm01_vnet1
ovs-vsctl set Interface hv1_vm01_vnet1 external_ids:iface-id=hv1_vm01_vnet1

sleep 5
dmesg -C

# qos
ovs-vsctl set interface $port2 external-ids:ovn-egress-iface=true
#ovn-nbctl set Logical_Switch_Port ln_p1 options:qos_min_rate=10000000
#ovn-nbctl set Logical_Switch_Port ln_p1 options:qos_max_rate=20000000
#ovn-nbctl set Logical_Switch_Port ln_p1 options:qos_burst=22000000
#echo "#### check 10M - 20M"
#tc class show dev $port2
#tc class show dev $port2|grep rate.*10M.*ceil.*20M || echo "BUG: 10M 20M"

ovn-nbctl set Logical_Switch_Port ln_p1 options:qos_min_rate=100000000
ovn-nbctl set Logical_Switch_Port ln_p1 options:qos_max_rate=200000000
ovn-nbctl set Logical_Switch_Port ln_p1 options:qos_burst=220000000
echo "#### check 100M - 200M"
tc class show dev $port2
tc class show dev $port2|grep rate.*100M.*ceil.*200M || echo "BUG: 100M 200M"
```
