---
layout: article
tags: KVM
title: VM Migration
mathjax: true
key: Linux
---

[redhat doc](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-kvm_live_migration-shared_storage_example_nfs_for_a_simple_migration)
{:.info}

## shared storage setup
```
虚拟机迁移需要用到额外的共享存储。
迁移的过程中，源端把虚拟机内存和设备信息拷贝到共享存储上，目的端再从共享存储中获取。
可以搭建一个nfs服务器，让nfs服务器export /var/lib/libvirt/images,然后把迁移源端和目的端的/var/lib/libvirt/images目录都mount到nfs服务器。
这里面/var/lib/libvirt/images是创建虚拟机时指定的disk path，也可以使用其他路径。
因为机器有限，我这里把nfs服务放到源端。

# 启动nfs
[root@hp-dl388g8-22 ~]# echo "/var/lib/libvirt/images *.redhat.com(rw,no_root_squash,sync)" > /etc/exports
[root@hp-dl388g8-22 ~]# systemctl restart nfs-server

# mount到nfs
[root@hp-dl380pg8-15 ovn]# mount hp-dl388g8-22.rhts.eng.pek2.redhat.com:/var/lib/libvirt/images /var/lib/libvirt/images

# 最好关闭selinux
# setenforce 0
# systemctl --now disable firewalld
# iptables -F
# ip6tables -F
```


## install vm

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

# when migrate, the cpu model should be the same and be supported by host
# use SandyBridge-IBRS that is supported by most latest intel cpu
virt-install \
        --name $vm_name \
        --vcpus=2 \
        --ram=2048 \
        --disk path=/var/lib/libvirt/images/$vm_name.qcow2,device=disk,bus=virtio,format=qcow2,cache=none \
        --network bridge=virbr0,model=virtio,mac=$mac4vm \
        --boot hd \
        --accelerate \
        --force \
        --graphics vnc,listen=0.0.0.0 \
        --os-type=linux \
        --cpu=SandyBridge-IBRS \
        --noautoconsole

```

## disable NM in VM
```
systemctl stop NetworkManager;iptables -F;ip6tables -F;setenforce 0;systemctl stop firewalld
```


## migrate
```
virsh migrate --unsafe --live --undefinesource --persistent v0 qemu+ssh://hp-dl380pg8-15.rhts.eng.pek2.redhat.com/system
```

## 运行迁移过来的虚拟机
我这里迁移完毕之后，需要在目的端把qcow2文件拷贝出来，
再umount /var/lib/libvirt/miages，  
再把qcow2文件拷贝到/var/lib/lib/images中，
然后重启迁移过来的虚拟机，才能让虚拟机正常运行，否则不能登录。
就是说需要保证qcow2文件存放在本地才行。

有可能disable NM in VM之后就没上面的问题了。
{:.info}
```
virsh destroy v0
umount /var/lib/libvirt/images/
cp /root/v0.qcow2 /var/lib/libvirt/images/
virsh start v0
virsh console v0

# 再迁移回去
mount hp-dl388g8-22.rhts.eng.pek2.redhat.com:/var/lib/libvirt/images /var/lib/libvirt/images
virsh migrate --unsafe --live --undefinesource --persistent v0 qemu+ssh://hp-dl388g8-22.rhts.eng.pek2.redhat.com/system

```
