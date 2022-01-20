---
layout: article
tags: Linux
title: kernel options
mathjax: true
key: Linux
---

[https://beaker-project.org/docs/](https://beaker-project.org/docs/)
{:.info} 

## modprobe.blacklist

```
You can use any of the below method to enable qedf debug logs.
echo 0x1 > /sys/module/qedf/parameters/debug
insmod drivers/scsi/qedf/qedf.ko debug=1

I installed a distro first with kernel option modprobe.blacklist=qedf.
After the system boot up, i installed your kernel with "rpm -ivh --force",
then i removed modprobe.blacklist=qedf from kernel options and 
enabled qedf debug option by writing a conf file to /etc/modprobe.d/,
then i restarted system, and the issue happened during rebooting.
```

## Differences between modprobe.blacklist and rd.driver.blacklist in Linux kernel parameters
```
rd.driver.blacklist is a configuration option for the kernel command line, 
to be applied when the kernel is loaded from the Linux boot image ( the initramfs ). 
Here you would call out certain kernel modules that are loaded from that initial filesystem image.

Later on, after the root filesystem is online i.e. has been mounted and the operating system is loading loadable modules ... 
you can use modprobe.blacklist to affect the handling of loadable modules. 
modprobe actually reads the kernel command line, to find parameters that affect loadable modules. 
So while it looks like this parameter is applicable to loading of the Linux kernel, it is not really. 
modprobe finds it and uses modprobe.blacklist along with other loadable module parameters.

So whether to use the ramdisk option, or the modprobe option ... 
depends on whether the driver in question resides in the boot image ( put there by dracut ), 
or resides in the root filesystem of the OS ( and is handled by modprobe ).


```

## How do I prevent a kernel module from loading automatically?
```
https://access.redhat.com/solutions/41278

[ step1 ] First we unload the module from the running system if it is loaded.
# modprobe -r module_name   

[ step2 ] To prevent a module from being loaded directly 
you add the blacklist line to a configuration file specific to the system configuration 
-- for example /etc/modprobe.d/local-dontload.conf.

This alone will not prevent a module being loaded if it is a required or an optional dependency of another module. 
Some kernel modules will attempt to load optional modules on demand, which we mitigate in the step3.
# echo "blacklist module_name" >> /etc/modprobe.d/local-dontload.conf            #step2

[ step3 ] The install line simply causes /bin/false to be run instead of installing a module. (The same can be achieved by using /bin/true.)
The next time the loading of the module is attempted, the /bin/false will be executed instead. 
This will prevent the module from being loaded on-demand. 
If the excluded module is required for other specific hardware, there might be unexpected side effects.
# echo "install module_name /bin/false" >> /etc/modprobe.d/local-dontload.conf   #step3

Now continue with the relevant steps for your system's version of RHEL:

Finishing Steps for Red Hat Enterprise Linux 8 only.

[ step4 ] Make a backup copy of your initramfs.
# cp /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.$(date +%m-%d-%H%M%S).bak

[ step5 ] If the kernel module is part of the initramfs (use "lsinitrd /boot/initramfs-$(uname -r).img|grep module-name.ko" to verify), 
then you should rebuild your initial ramdisk image, 
omitting the module to be avoided (see How to rebuild the initial ramdisk image in Red Hat Enterprise Linux for more information).
# dracut --omit-drivers module_name -f

[ step6 ] Get the current kernel command line parameters.
# grub2-editenv - list | grep kernelopts

[ step7 ] Append module_name.blacklist=1 rd.driver.blacklist=module_name at the end of the output found in #step 6. 
(For RHEl 8, grub2-editenv utility is actually the recommended method for altering these variables.)
# grub2-editenv - set kernelopts="<output-from-step-6> module_name.blacklist=1 rd.driver.blacklist=module_name"

For example: 
# grub2-editenv - set kernelopts="root=/dev/mapper/rhel_example-root ro crashkernel=auto resume=/dev/mapper/rhel_example-swap rd.lvm.lv=rhel_example/root rd.lvm.lv=rhel_example/swap module_name.blacklist=1 rd.driver.blacklist=module_name"

[ step8 ] Make a backup copy of the kdump initramfs.
# cp /boot/initramfs-$(uname -r)kdump.img /boot/initramfs-$(uname -r)kdump.img.$(date +%m-%d-%H%M%S).bak

[ step9 ] Append rd.driver.blacklist=module_name to the KDUMP_COMMANDLINE_APPEND setting in /etc/sysconfig/kdump. 
This will cause it to be omitted from the kdump initramfs.
# sed -i '/^KDUMP_COMMANDLINE_APPEND=/s/"$/ rd.driver.blacklist=module_name"/' /etc/sysconfig/kdump

[ step10 ] Restart the kdump service to pick up the changes to kdump's initrd.
# kdumpctl restart

[ step11 ] Rebuild the kdump initial ramdisk image.
# mkdumprd -f /boot/initramfs-$(uname -r)kdump.img

[ step12 ] Reboot the system at a convenient time to have the changes take effect.
# reboot

```

```
I suggest to add the modules to a separate blacklist configuration file, e.g. /etc/modprobe.d/myblacklist.conf 
to have a clear distinction between the own blacklisted modules and the distribution shipped configuration file.

Also, after adding the module, the initramfs has to be regenerated to prevent the kernel module to be loaded during the initramfs phase, 
should the module be part of the initramfs image. To regenerate the initramfs, the user has to issue:

# dracut -f

To prevent the module to be loaded in the initramfs, without regenerating it, 
the kernel command line parameter rdblacklist=<module name>(rd.driver.blacklist) can be used on the kernel command line for RHEL-6.

For RHEL-7 the kernel command line parameter modprobe.blacklist=<module name> can be used to 
blacklist the module for the initramfs as well as the real root, 
without the need to create a modprobe.d configuration file and regeneration of the initramfs.
```

```
I have just checked the option above and the dracut option was not required . 
However you will need to make sure that If the module you are blacklisting is a dependency of another module, the blacklsting will fail. 
It will be still loaded.
```
