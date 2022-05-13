---
layout: article
tags: OVN
title: OVN - External Port
mathjax: true
key: Linux
---

External Port可以为外部端口提供DNS等服务。

## install vms
install two vms on both node
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

## setup on central node
```
systemctl start openvswitch
systemctl start ovn-northd
ovn-sbctl set-connection ptcp:6642
ovn-nbctl set-connection ptcp:6641

ovs-vsctl set Open_vSwitch . external-ids:system-id=hv0
ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:177.1.1.1:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=177.1.1.1
systemctl restart ovn-controller


mac_v0_vnet1=04:ac:10:ff:01:94
mac_v0_vnet2=04:ac:10:ff:01:95
mac_v1_vnet1=04:ac:10:ff:01:96
mac_v1_vnet2=04:ac:10:ff:01:97

cat <<-EOF > v0-vnet1.xml 
<interface type='bridge'>
        <target dev='h0_v0_vnet1'/>
        <mac address='${mac_v0_vnet1}'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>
EOF


cat <<-EOF > v0-vnet2.xml 
<interface type='bridge'>
        <target dev='h0_v0_vnet2'/>
        <mac address='${mac_v0_vnet2}'/>
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

cat <<-EOF > v1-vnet2.xml 
<interface type='bridge'>
        <target dev='h0_v1_vnet2'/>
        <mac address='${mac_v1_vnet2}'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>
EOF

#virsh attach-device v0 v0-vnet1.xml
virsh attach-device v1 v1-vnet1.xml

#ovs-vsctl set interface h0_v0_vnet1 external-ids:iface-id=h0_v0_vnet1
ovs-vsctl set interface h0_v1_vnet1 external-ids:iface-id=h0_v1_vnet1

```

## setup on computing node 
```
systemctl start openvswitch
ovs-vsctl set Open_vSwitch . external-ids:system-id=hv1
ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:177.1.1.1:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=177.1.1.2
systemctl start ovn-controller

mac_v0_vnet1=02:ac:10:ff:01:94
mac_v0_vnet2=02:ac:10:ff:01:95
mac_v1_vnet1=02:ac:10:ff:01:96
mac_v1_vnet2=02:ac:10:ff:01:97

cat <<-EOF > v0-vnet1.xml
<interface type='bridge'>
        <target dev='h1_v0_vnet1'/>
        <mac address='${mac_v0_vnet1}'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>
EOF


cat <<-EOF > v0-vnet2.xml
<interface type='bridge'>
        <target dev='h1_v0_vnet2'/>
        <mac address='${mac_v0_vnet2}'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'/>
        <model type='virtio'/>
</interface>
EOF

cat <<-EOF > v1-vnet1.xml
<interface type='bridge'>
        <target dev='h1_v1_vnet1'/>
        <mac address='${mac_v1_vnet1}'/>
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

## create topo on central node
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
ovn-nbctl ls-add ls2 -- add Logical_Switch ls2 other_config subnet=172.16.2.0/24

# setup ls ipv6_prefix
ovn-nbctl set Logical-switch ls1 other_config:ipv6_prefix=2001:db8:1::0
ovn-nbctl set Logical-switch ls2 other_config:ipv6_prefix=2001:db8:2::0

# create dhcp_options
dhcp_options1=$(ovn-nbctl create DHCP_Options cidr=172.16.1.0/24 \
        options="\"server_id\"=\"172.16.1.254\" \"server_mac\"=\"00:00:00:00:01:00\" \
        \"lease_time\"=\"$((36000 + RANDOM % 3600))\" \"router\"=\"172.16.1.254\" \"dns_server\"=\"172.16.1.254\"")
dhcp_options2=$(ovn-nbctl create DHCP_Options cidr=172.16.2.0/24 \
        options="\"server_id\"=\"172.16.2.254\" \"server_mac\"=\"00:00:00:00:02:00\" \
        \"lease_time\"=\"$((36000 + RANDOM % 3600))\" \"router\"=\"172.16.2.254\" \"dns_server\"=\"172.16.2.254\"")

dhcpv6_options1=$(ovn-nbctl create DHCP_Options cidr="2001\:db8\:1\:\:0/64" \
                                options="\"server_id\"=\"00:00:00:00:01:00\" \"dns_server\"=\"2001:db8:1::254\"")
dhcpv6_options2=$(ovn-nbctl create DHCP_Options cidr="2001\:db8\:2\:\:0/64" \
                                options="\"server_id\"=\"00:00:00:00:02:00\" \"dns_server\"=\"2001:db8:2::254\"")

# create logical switch port and setup dhcp_option

#lsp_name=h0_v0_vnet1
#mac=$mac_h0_v0_vnet1
#ovn-nbctl lsp-add ls1 $lsp_name
#ovn-nbctl lsp-set-addresses $lsp_name "$mac dynamic"
#ovn-nbctl lsp-set-dhcpv4-options $lsp_name ${dhcp_options1}
#ovn-nbctl add Logical_Switch_Port $lsp_name dhcpv6_options ${dhcpv6_options1}

lsp_name=h0_v1_vnet1
mac=$mac_h0_v1_vnet1
ovn-nbctl lsp-add ls1 $lsp_name
ovn-nbctl lsp-set-addresses $lsp_name "$mac dynamic"
ovn-nbctl lsp-set-dhcpv4-options $lsp_name ${dhcp_options1}
ovn-nbctl add Logical_Switch_Port $lsp_name dhcpv6_options ${dhcpv6_options1}

lsp_name=h1_v0_vnet2
mac=$mac_h1_v0_vnet2
ovn-nbctl lsp-add ls2 $lsp_name
ovn-nbctl lsp-set-addresses $lsp_name "$mac dynamic"
ovn-nbctl lsp-set-dhcpv4-options $lsp_name ${dhcp_options2}
ovn-nbctl add Logical_Switch_Port $lsp_name dhcpv6_options ${dhcpv6_options2}

lsp_name=h1_v1_vnet2
mac=$mac_h1_v1_vnet2
ovn-nbctl lsp-add ls2 $lsp_name
ovn-nbctl lsp-set-addresses $lsp_name "$mac dynamic"
ovn-nbctl lsp-set-dhcpv4-options $lsp_name ${dhcp_options2}
ovn-nbctl add Logical_Switch_Port $lsp_name dhcpv6_options ${dhcpv6_options2}

# create logical router lr1
# There are two way to create l3 gateway
# The one is : ovn-nbctl create Logical_Router name=lr1 options:chassis={chassis_uuid}
# and the other one is: ovn-nbctl lrp-set-gateway-chassis lr1-ls3 dfe51fb0-0c76-437c-ad95-709b98998b4f
ovn-nbctl lr-add lr1
#ovn-nbctl create Logical_Router name=lr1 options:chassis=hv0
ovn-nbctl lrp-add lr1 lr1-ls1 00:00:00:00:01:00 172.16.1.254/24
ovn-nbctl lrp-add lr1 lr1-ls2 00:00:00:00:02:00 172.16.2.254/24

# connect lr1 and ls1
ovn-nbctl lsp-add ls1 ls1-lr1
ovn-nbctl lsp-set-type ls1-lr1 router
ovn-nbctl lsp-set-addresses ls1-lr1 00:00:00:00:01:00
ovn-nbctl lsp-set-options ls1-lr1 router-port=lr1-ls1

# connect lr1 and ls2
ovn-nbctl lsp-add ls2 ls2-lr1
ovn-nbctl lsp-set-type ls2-lr1 router
ovn-nbctl lsp-set-addresses ls2-lr1 00:00:00:00:02:00
ovn-nbctl lsp-set-options ls2-lr1 router-port=lr1-ls2

# connect to outside

#ovn-nbctl ls-add ls3
#ovn-nbctl lrp-add lr1 lr1-ls3 00:00:00:00:03:00 172.16.3.254/24
#ovn-nbctl lsp-add ls3 ls3-lr1
#ovn-nbctl lsp-set-type ls3-lr1 router
#ovn-nbctl lsp-set-addresses ls3-lr1 00:00:00:00:03:00
#ovn-nbctl lsp-set-options ls3-lr1 router-port=lr1-ls3

ovn-nbctl lsp-add ls1 ls1-localnet
ovn-nbctl lsp-set-type ls1-localnet localnet
ovn-nbctl lsp-set-addresses ls1-localnet unknown
ovn-nbctl lsp-set-options ls1-localnet network_name=outNet

ovs-vsctl add-br br-out
ovs-vsctl add-port br-out enp5s0f1
ovs-vsctl set Open_vSwitch . external-ids:ovn-bridge-mappings=outNet:br-out


# attach vf to v0
domain=0000
bus=05
slot=10
function=0

cat <<- EOF > sriov.xml
                <interface type='hostdev' managed='yes'>
                        <source>
                                <address type='pci' domain='0x${domain}' bus='0x${bus}' slot='0x${slot}' function='0x${function}'/>
                        </source>
                        <mac address='00:00:00:02:04:06'/>
                </interface>
EOF

virsh attach-device v0 sriov.xml


# add ls1-lp-ext1 for external port
ovn-nbctl lsp-add ls1 ls1-lp-ext1
ovn-nbctl lsp-set-addresses ls1-lp-ext1 "00:00:00:02:04:06 172.16.1.11 2001:db8:1::11"
ovn-nbctl lsp-set-type ls1-lp-ext1 external
ovn-nbctl lsp-set-dhcpv4-options ls1-lp-ext1 ${dhcp_options1}

ovn-nbctl lsp-set-options ls1-lp-ext1 requested-chassis=hv0
ovn-nbctl lsp-set-options ls1-lp-ext1 requested-chassis=3b90fffc-7fd5-43ff-acdc-d19575b5d116

v0的mac和ovn指定的external port ls1-lp-ext1 mac 一样，
并且v0可以通过localnet接入ovn网络.
当v0的dhcp请求发送到ovn网络之后，
ovn发现有一个external port ls1-lp-ext1的mac和v0 mac匹配，所以可以服务该dhcp请求。
```

