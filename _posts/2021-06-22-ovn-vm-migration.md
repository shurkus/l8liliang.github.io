---
layout: article
tags: OVN
title: OVN - VM Migration
mathjax: true
key: Linux
---

[redhat doc](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-kvm_live_migration-shared_storage_example_nfs_for_a_simple_migration)
{:.info}

## setup on central node

```
# stop firewalld selinux...
iptables -F
ip6tables -F
setenforce 0
systemctl --now disable firewalld
#systemctl stop NetworkManager

# start ovn
systemctl start openvswitch
systemctl start ovn-northd
ovn-sbctl set-connection ptcp:6642
ovn-nbctl set-connection ptcp:6641
ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:177.1.1.1:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=177.1.1.1
systemctl start ovn-controller

# create shared storage for vm migration
mkdir -p /home/test1/images/ &>/dev/null
echo "/home/test1/images *.redhat.com(rw,no_root_squash,sync)" > /etc/exports
systemctl restart nfs-server

# install vms
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

rm -rf /home/test/images/*

# define vm name and mac
vm_name=v0
mac4vm=a4:a4:a4:a4:a4:a0
ip=192.168.122.10

# download image
wget http://netqe-bj.usersys.redhat.com/share/vms/rhel8.4.qcow2 -O /home/test1/images/$vm_name.qcow2

virsh net-update default add ip-dhcp-host "<host mac='$mac4vm' ip='$ip' />" --live --config

virt-install \
        --name $vm_name \
        --vcpus=2 \
        --ram=2048 \
        --disk path=/home/test1/images/$vm_name.qcow2,device=disk,bus=virtio,format=qcow2,cache=none \
        --network bridge=virbr0,model=virtio,mac=$mac4vm \
        --boot hd \
        --accelerate \
        --force \
        --graphics vnc,listen=0.0.0.0 \
        --os-type=linux \
        --cpu=SandyBridge-IBRS \
        --noautoconsole


# define vm name and mac
vm_name=v1
mac4vm=a4:a4:a4:a4:a4:a1
ip=192.168.122.11

# download image
wget http://netqe-bj.usersys.redhat.com/share/vms/rhel8.4.qcow2 -O /home/test1/images/$vm_name.qcow2

virsh net-update default add ip-dhcp-host "<host mac='$mac4vm' ip='$ip' />" --live --config

virt-install \
        --name $vm_name \
        --vcpus=2 \
        --ram=2048 \
        --disk path=/home/test1/images/$vm_name.qcow2,device=disk,bus=virtio,format=qcow2,cache=none \
        --network bridge=virbr0,model=virtio,mac=$mac4vm \
        --boot hd \
        --accelerate \
        --force \
        --graphics vnc,listen=0.0.0.0 \
        --os-type=linux \
        --cpu=SandyBridge-IBRS \
        --noautoconsole


mac_v0_vnet1=04:ac:10:ff:01:94
mac_v1_vnet1=04:ac:10:ff:01:96
uuid_vnet_0_1=$(uuidgen)
uuid_vnet_1_1=$(uuidgen)

# 注意： 一定要在xml中设置interfaceid，如果通过命令行set external-ids，
#        虚拟机迁移之后就ping不通
cat <<-EOF > v0-vnet1.xml
<interface type='bridge'>
        <target dev='h0_v0_vnet1'/>
        <mac address='${mac_v0_vnet1}'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'>
                <parameters interfaceid='${uuid_vnet_0_1}'/>
        </virtualport>
        <model type='virtio'/>
</interface>
EOF

cat <<-EOF > v1-vnet1.xml
<interface type='bridge'>
        <target dev='h0_v1_vnet1'/>
        <mac address='${mac_v1_vnet1}'/>
        <source bridge='br-int'/>
        <virtualport type='openvswitch'>
                <parameters interfaceid='${uuid_vnet_1_1}'/>
        </virtualport>
        <model type='virtio'/>
</interface>
EOF

vmsh run_cmd v0 "systemctl stop NetworkManager;iptables -F;ip6tables -F;setenforce 0;systemctl stop firewalld"
vmsh run_cmd v1 "systemctl stop NetworkManager;iptables -F;ip6tables -F;setenforce 0;systemctl stop firewalld"
virsh attach-device v0 v0-vnet1.xml
virsh attach-device v1 v1-vnet1.xml

# create topo
dhcp_options=$(ovn-nbctl create DHCP_Options cidr=172.16.1.0/24 \
        options="\"server_id\"=\"172.16.1.254\" \"server_mac\"=\"00:00:00:00:01:00\" \
        \"lease_time\"=\"$((36000 + RANDOM % 3600))\" \"router\"=\"172.16.1.254\"")

dhcpv6_options=$(ovn-nbctl create DHCP_Options cidr="2001\:db8\:1\:\:0/64" \
                options="\"server_id\"=\"00:00:00:00:01:00\"")

ovn-nbctl ls-add ls1

ovn-nbctl lsp-add ls1 ${uuid_vnet_0_1} -- lsp-set-dhcpv4-options ${uuid_vnet_0_1} $dhcp_options
ovn-nbctl lsp-set-addresses ${uuid_vnet_0_1} "04:ac:10:ff:01:94 172.16.1.2 2020:1::2"

ovn-nbctl lsp-add ls1 ${uuid_vnet_1_1} -- lsp-set-dhcpv4-options ${uuid_vnet_1_1} $dhcp_options
ovn-nbctl lsp-set-addresses ${uuid_vnet_1_1} "04:ac:10:ff:01:96 172.16.1.3 2020:1::3"
```


## setup on computing node
```
[root@hp-dl380pg8-15 migrate]# cat setup.sh 
# stop firewalld selinux...
iptables -F
ip6tables -F
setenforce 0
systemctl --now disable firewalld

# start ovn
systemctl start openvswitch
systemctl start ovn-northd
ovn-sbctl set-connection ptcp:6642
ovn-nbctl set-connection ptcp:6641

ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:177.1.1.1:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=177.1.1.2
systemctl start ovn-controller

# define default vnet
virsh net-list | grep -qe "^ default" || { virsh net-define /usr/share/libvirt/networks/default.xml; virsh net-start default; }

# mount shared storage
umount /home/test1/images &>/dev/null
rm -rf /home/test1/images/*
mkdir -p /home/test1/images &>/dev/null
mount hp-dl388g8-22.rhts.eng.pek2.redhat.com:/home/test1/images /home/test1/images

```

## migrate on central node
```

# get ip in vm before migration
[root@hp-dl388g8-22 migrate]# virsh console v0
[root@localhost ~]# dhclient -v ens7
[root@hp-dl388g8-22 migrate]# virsh console v1
[root@localhost ~]# dhclient -v ens7
[root@localhost ~]# ping 172.16.1.2
PING 172.16.1.2 (172.16.1.2) 56(84) bytes of data.
64 bytes from 172.16.1.2: icmp_seq=1 ttl=64 time=2.37 ms
64 bytes from 172.16.1.2: icmp_seq=2 ttl=64 time=0.403 ms

# migrate
[root@hp-dl388g8-22 migrate]# virsh migrate --unsafe --live --undefinesource --persistent v0 qemu+ssh://hp-dl380pg8-15.rhts.eng.pek2.redhat.com/system

# ping after migration
[root@hp-dl388g8-22 migrate]# virsh console v1
Connected to domain v1
Escape character is ^]

[root@localhost ~]# ping 172.16.1.2
PING 172.16.1.2 (172.16.1.2) 56(84) bytes of data.
64 bytes from 172.16.1.2: icmp_seq=1 ttl=64 time=1.93 ms
64 bytes from 172.16.1.2: icmp_seq=2 ttl=64 time=0.686 ms
64 bytes from 172.16.1.2: icmp_seq=3 ttl=64 time=0.662 ms
64 bytes from 172.16.1.2: icmp_seq=4 ttl=64 time=0.679 ms

```


## ovn clenup
```
ovn_cleanup()
{
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

        [ x"TUNNEL_IF_TYPE" == x"dpdk" ] && dpdk-devbind -b $NIC_DRIVER $dpdk0

        virsh list --all | sed -n 3~1p | awk '/[[:alpha:]]+/ {
                if ($3 == "running") {
                        system("virsh shutdown "$2" &>/dev/null");
                        sleep 2;
                        system("virsh destroy "$2" &>/dev/null")
                };
                system("virsh undefine --managed-save --snapshots-metadata --remove-all-storage "$2" &>/dev/null")
        }'
        virsh net-list --all | sed -n 3~1p | awk '/[[:alnum:]]+/ {
                system("virsh net-destroy "$1" &>/dev/null");
                sleep 2;
                system("virsh net-undefine "$1" &>/dev/null")
        }'

}
```
