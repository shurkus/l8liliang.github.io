---
layout: article
tags: Linux
title: Execute Script On Boot
mathjax: true
key: Linux
---

## Steps
```
1, create your srcipt under /usr/local/bin
[root@dhcp-128-17 ~]# cat /usr/local/bin/start_job_scheduler.sh
#!/bin/bash

klogin liali 123456
pgrep watchdog.sh || { cd /var/www/html/job_scheduler; ./watchdog.sh & }
pgrep ci-manager.sh || { cd /var/www/html/job_scheduler/ci; ./ci-manager.sh &}

2, call your script in /etc/rc.d/rc.local
[root@dhcp-128-17 ~]# cat /etc/rc.d/rc.local
#!/bin/sh -e

echo 111 >> /home/lll
su - liliang -l -c "start_job_scheduler.sh"

3, create symbolic link to rc.local
ln -s /etc/rc.d/rc5.d/S99local /etc/rc.d/rc.local

4, add execute permission
chmod +x /etc/rc.d/rc.local
chmod +x /usr/local/bin/start_job_scheduler.sh

```
