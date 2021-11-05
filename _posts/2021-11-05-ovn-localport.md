---
layout: article
tags: OVN
title: OVN - localport
mathjax: true
key: Linux
---

## localport
```
ovn 中localport的端口代表本地端口，这种端口只能与本chassis中的其他端口通信.
另外也可以通过localnet与external port通信。

A localport logical switch port is a special kind of VIF logical switch
       port.  These  ports are present in every chassis, not bound to any par‐
       ticular one. Traffic to such a port will never be forwarded  through  a
       tunnel, and traffic from such a port is expected to be destined only to
       the same chassis, typically in response to a request it received. Open‐
       Stack  Neutron  uses a localport port to serve metadata to VMs. A meta‐
       data proxy process is attached to this port on every host and  all  VMs
       within  the same network will reach it at the same IP/MAC address with‐
       out any traffic being sent over a tunnel. For further details, see  the
       OpenStack documentation for networking-ovn.

```

## The localport should not leaks traffic to the provider network it is attached to
```

FROM: https://bugzilla.redhat.com/show_bug.cgi?id=1939470#c14

#!/bin/bash

systemctl start openvswitch                                                
systemctl start ovn-northd                                
ovn-nbctl set-connection ptcp:6641                                                          
ovn-sbctl set-connection ptcp:6642                                    
ovs-vsctl set open . external_ids:system-id=hv1 external_ids:ovn-remote=tcp:20.0.175.25:6642 external_ids:ovn-encap-type=geneve external_ids:ovn-encap-ip=20.0.175.25
systemctl restart ovn-controller                                      
                                                                
# provider network
ovs-vsctl add-br br-phys
ip link set br-phys up
ovs-vsctl set open . external-ids:ovn-bridge-mappings=phys:br-phys

# create lp,ln,lsp
ovn-nbctl ls-add ls \
    -- lsp-add ls lp \
    -- lsp-set-type lp localport \
    -- lsp-set-addresses lp "00:00:00:00:00:01 10.0.0.1" \
    -- lsp-add ls ln \
    -- lsp-set-type ln localnet \
    -- lsp-set-options ln network_name=phys \
    -- lsp-add ls lsp \
    -- lsp-set-addresses lsp "00:00:00:00:00:02 10.0.0.2"

# create "vm" for lp
ovs-vsctl add-port br-int lp -- set interface lp type=internal external_ids:iface-id=lp
ip netns add lp
ip link set lp netns lp
ip netns exec lp ip link set lp address 00:00:00:00:00:01
ip netns exec lp ip link set lp up
ip netns exec lp tcpdump -i lp -w lp.pcap &
ip netns exec lp ip addr add 10.0.0.1/24 dev lp

ovn-nbctl --wait=hv sync

# create "vm" for lsp
ovs-vsctl add-port br-int lsp -- set interface lsp type=internal external_ids:iface-id=lsp options:tx_pcap=lsp.pcap options:rxq_pcap=lsp-rx.pcap
ip netns add lsp
ip link set lsp netns lsp
ip netns exec lsp ip link set lsp address 00:00:00:00:00:02
ip netns exec lsp ip link set lsp up
# tcpdump on lsp
ip netns exec lsp tcpdump -i lsp -w lsp.pcap &
ip netns exec lsp ip addr add 10.0.0.2/24 dev lsp

ovn-nbctl --wait=hv sync
sleep 30
pkill tcpdump
sleep 2
tcpdump -r lsp.pcap -nnle -v arp
tcpdump -r lp.pcap -nnle -v arp

# The localport traffic should not be leaked to lsp through localnet.
```


## But the localport traffic should can reach external port through localnet
```

FROM: https://bugzilla.redhat.com/show_bug.cgi?id=1974062#c14

# instead of dropping all traffic coming from localport to localnet, drop all traffic EXCEPT traffic directed to an external port.

systemctl start openvswitch          
systemctl start ovn-northd                                          
ovn-nbctl set-connection ptcp:6641
ovn-sbctl set-connection ptcp:6642
ovs-vsctl set open . external_ids:system-id=hv1 external_ids:ovn-remote=tcp:1.1.172.25:6642 external_ids:ovn-encap-type=geneve external_ids:ovn-encap-ip=1.1.172.25
systemctl restart ovn-controller                                             
                                                    
ovs-vsctl add-br br-phys                                                   
ip link set br-phys up  

# provider network                              
ovs-vsctl set open . external-ids:ovn-bridge-mappings=phys:br-phys
                                 
ovn-nbctl ls-add ls                                                                    

# create external port 1                                           
ovn-nbctl --wait=sb ha-chassis-group-add hagrp
ovn-nbctl --wait=sb ha-chassis-group-add-chassis hagrp hv1 10         
ovn-nbctl lsp-add ls lext                                               
ovn-nbctl lsp-set-addresses lext "00:00:00:00:00:04 10.0.0.4 2001::4"  
ovn-nbctl lsp-set-type lext external                                     
hagrp_uuid=`ovn-nbctl --bare --columns _uuid find ha_chassis_group name=hagrp`
ovn-nbctl set logical_switch_port lext ha_chassis_group=$hagrp_uuid

# create lp,lsp                                             
ovn-nbctl lsp-add ls lp \                                                                                                                       
    -- lsp-set-type lp localport \
    -- lsp-set-addresses lp "00:00:00:00:00:01 10.0.0.1 2001::1" \
    -- lsp-add ls lsp \                                    
    -- lsp-set-addresses lsp "00:00:00:00:00:02 10.0.0.2 2001::2"

# create external port 2 (looks like this is unneeded)                                                 
ovn-nbctl lsp-add ls lext2                      
ovn-nbctl lsp-set-addresses lext2 "00:00:00:00:00:10 10.0.0.10 2001::10"
ovn-nbctl lsp-set-type lext2 external          
ovn-nbctl set logical_switch_port lext2 ha_chassis_group=$hagrp_uuid
ovn-nbctl --wait=hv sync                       

# create external and delete it                                               
ovn-nbctl lsp-add ls lext-deleted     
ovn-nbctl lsp-set-addresses lext-deleted "00:00:00:00:00:03 10.0.0.3 2001::3"
ovn-nbctl lsp-set-type lext-deleted external        
ovn-nbctl set logical_switch_port lext-deleted ha_chassis_group=$hagrp_uuid
ovn-nbctl --wait=hv sync
ovn-nbctl lsp-del lext-deleted
ovn-nbctl --wait=hv sync

# create "vm" for lp
ovs-vsctl add-port br-int lp -- set interface lp type=internal external_ids:iface-id=lp
ip netns add lp
ip link set lp netns lp
ip netns exec lp ip link set lp address 00:00:00:00:00:01
ip netns exec lp ip link set lp up
ip netns exec lp ip addr add 10.0.0.1/24 dev lp
ip netns exec lp ip addr add 2001::1/64 dev lp

# create "vm" for lsp
ovn-nbctl --wait=hv sync
ovs-vsctl add-port br-int lsp -- set interface lsp type=internal external_ids:iface-id=lsp options:tx_pcap=lsp.pcap options:rxq_pcap=lsp-rx.pcap
ip netns add lsp
ip link set lsp netns lsp
ip netns exec lsp ip link set lsp address 00:00:00:00:00:02
ip netns exec lsp ip link set lsp up
ip netns exec lsp ip addr add 10.0.0.2/24 dev lsp
ip netns exec lsp ip addr add 2001::2/64 dev lsp
ip netns exec lsp tcpdump -i lsp -w lsp.pcap &

# create a outside "host"
ovs-vsctl add-port br-phys ext1 -- set interface ext1 type=internal
ip netns add ext1
ip link set ext1 netns ext1
ip netns exec ext1 ip link set ext1 up
ip netns exec ext1 ip addr add 10.0.0.101/24 dev ext1
ip netns exec ext1 ip addr add 2001::101/64 dev ext1
ip netns exec ext1 tcpdump -i ext1 -w ext1.pcap &
sleep 2

# connect outside through provider network "phys"
ovn-nbctl lsp-add ls ln \
    -- lsp-set-type ln localnet \
    -- lsp-set-addresses ln unknown \
    -- lsp-set-options ln network_name=phys

# prepare neighbour info and ping   
ip netns exec lp ip neigh add 10.0.0.4 lladdr 00:00:00:00:00:04 dev lp
ip netns exec lp ip -6 neigh add 2001::4 lladdr 00:00:00:00:00:04 dev lp
ip netns exec lp ip neigh add 10.0.0.10 lladdr 00:00:00:00:00:10 dev lp
ip netns exec lp ip -6 neigh add 2001::10 lladdr 00:00:00:00:00:10 dev lp
ip netns exec lp ping 10.0.0.4 -c 1 -w 1 -W 1
ip netns exec lp ping 10.0.0.10 -c 1 -w 1 -W 1
ip netns exec lp ping6 2001::4 -c 1 -w 1 -W 1
ip netns exec lp ping6 2001::10 -c 1 -w 1 -W 1
sleep 1
pkill tcpdump
sleep 1
tcpdump -r ext1.pcap -nnle

# ping to external port should success

```
