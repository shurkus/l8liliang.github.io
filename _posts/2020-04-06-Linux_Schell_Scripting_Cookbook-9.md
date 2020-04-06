---
layout: article
tags: Shell
title: Linux Shell脚本攻略 -- 系统监控
mathjax: true
key: Shell
---

## du (disk usage)

```
du可以查看文件或目录占用的磁盘空间

du file1 file2
du -a directory   //递归输出目录统计结果
du -h             //以KB MB为单位显示磁盘使用情况
du -c file1 file2 //给出两个文件的和
du -s dir         //给出当前目录下所有文件的磁盘使用和
du -b             //以字节为单位
du -k             //以KB为单位
du -m             //以MB为单位
du -B BLOCK_SIZE  //以BLOCK_SIZE为单位
du --exclude "WILDCARD" dir
du --exclude-from filename
du --max-depth 2 dir

# 找出4个最大的文件，包括目录
du -ak /home | sort -nrk 1 | head -n 4
bogon:Network_Demo liliang$ du -ak ~/Leon/Git | sort -nrk 1 | head -n 4
733964	/Users/liliang/Leon/Git
338284	/Users/liliang/Leon/Git/l8liliang.github.io
327448	/Users/liliang/Leon/Git/Auto-Layout-Demystified
293468	/Users/liliang/Leon/Git/Auto-Layout-Demystified/2ndEdition

# 找出4个最大的文件，不包括目录
find . -type f -exec du -k {} \; | sort -nrk 1 | head -n 4
```

## df (disk free)
```
df -h
df -h /home/user
```

## time
```
time可以测量应用程序的执行时间
time APPLICATION
time会执行APPLICATION，当执行完毕后，time将器real时间、sys时间、user时间输出到stderr中，将APPLICATION的正常输出发送到stdout。
time有一个shell内建到命令了，也有一个二进制文件位于/usr/bin/time，默认使用shell内建命令。但是内建到命令选项有限。建议使用绝对路径。
/usr/bin/time -o output.txt COMMAND    //-o可以把时间信息写到文件中
/usr/bin/time -a -o output.txt COMMAND //-a与-o配合，表示把时间信息追加到文件中

# 使用-f可以格式化输出
%e real时间，就是开始到结束的时间
%E real时间，就是开始到结束的时间
%U 用户模式下花费的时间
%S 内核模式下花费的时间
%Z 以字节为单位的系统页面大小
%C 命令名称
%D 进程非共享数据区的平均大小，KB
%x 命令的退出状态
%k 进程接收的信号数量
%W 进程被交换出主寸的次数
%P 进程所获得cpu时间百分比。等于(user+system)/real
%K 进程平均内存使用量 KB
%w 进程主动进行上下文切换的次数，例如等等IO操作完成
%c 进程被迫进行上下文切换的次数，由于时间片到期
```

## who, w, users, uptime, last, lastb
```
who命令获取当前在线用户信息
w命令获取当前在线用户的更详细的信息
users列出当前在线用户列表
uptime可以查看系统加电运行时长
last命令可以获取自文件/var/log/wtmp创建之后登录过系统的用户列表,也可以针对某个用户进行查询(last USER)
lastb可以获取失败的用户登录会话信息；lastb -F可以输出完整日期
```

## 列出一个小时内占用cpu最多的进程
```
#!/bin/bash

SECS=3600
UNIT_TIME=60

STEPS=$(( $SECS / $UNIT_TIME ))

echo Watching CPU Usage...

for((i=0;i<STEPS;i++))
do
        ps -eocomm,pcpu | egrep -v '(0.0)|(%CPU)' >> /tmp/cpu_usage.$$
        sleep $UNIT_TIME
done

echo CPU eaters :
cat /tmp/cpu_usage.$$ | awk '{ process[$1]+=$2; }
     END{
          for(i in process)
          {
                printf("%-20s %s\n",i,process[i]);
          }
         }' | sort -nrk 2 | head

rm /tmp/cpu_usage.$$
```

## watch监视命令输出
```
watch命令按照指定的时间间隔来执行命令并显示输出
watch -n 5 'ls -l'    
watch -n 30 -d 'commands' // -n表示间隔的秒数，-d表示着重标记输出中的差异
```

## inotifywait可以监视文件或目录并报告何时发生了某种事件
```
inotifywati -m -r -e create,move,delete $path -q

-m表示持续监视
-r递归监视目录（忽略符号连接）
-e指定需要监视的事件列表，access modify attrib move create open close delete
-q减少冗余信息
```

## 磁盘检测
```
#smartmontools
smartctl -a /dev/sda   // -a 包含设备的全部状态信息包括错误计数，准备时间，加电时间
smartctl -t /dev/sda   // 自检

#hdparm能给出更多数据
hdparm -I /dev/sda   //基本数据

hardparm命令可以测试磁盘性能。选项-t和-T分别测试缓冲(buffer)和缓存(cache)读操作。
hdparm -t /dev/sda
hdparm -T /dev/sda
```
