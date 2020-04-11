---
layout: article
tags: Shell
title: Linux Shell脚本攻略 -- 系统管理
mathjax: true
key: Shell
---

## ps

```
ps命令默认只显示当前终端开启的进程。
使用-e或者-A可以列出其他终端的进程。
使用-f可以列出更多列的信息。
使用-o可以指定显示哪些数据。
ps -eo comm,pcpu | head -5
pcpu   --> cpu占用率
pid    --> 进程ID
ppid   --> 父进程ID
pmem   --> 内存使用率
comm   --> 可执行文件名
cmd    --> 简单命令
user   --> 启动进程的用户
nice   --> 优先级
time   --> 累计cpu时间
etime  --> 进程启动后运行的时长
tty    --> 所关联的TTY设备
euid   --> 有效有话ID
stat   --> 进程状态

# ps e 可以显示进程的环境变量
ps -eo pid,cmd e | tail -n 1

# ps f 可以显示进程树
ps $pic f

# 对输出排序
ps -e --sort -parameter1,+parameter2
+表示生序； -表示降序
ps -eo comm,pcpu --sort -pcpu | head -5 //列出占用cpu最多的5个进程

# 根据用户过滤
-u  指定有效用户id或name，有效用户指的是当前进程使用的哪个用户的文件访问权限
-U  指定真实用户id或name，真实用户指的是创建进程的用户

# 根据TTY过滤
ps -t pts/0,pts/1

# 线程相关信息
-L可以显示出线程相关信息，该选项会在输出中添加一列LWP。
如果再加上-F选项，就会多出两列：NLWP（线程数量），LWP（线程ID）

ps -LF

# 根据命令找进程
ps -C bash -o pid=
如果在pid后面加上=，就会去掉ps输出中PID一列的列名，就是不显示表头的意思吧。

# pgrep可以使用-d指定输出分隔符，默认是换行
pgrep bash -d ":"
```

## 信号
```
信号是一种进程间通信机制。当进程收到一个信号时，它会执行对应的信号处理程序。
使用系统调用kill生成信号。trap命令可以在脚本中设置信号处理程序。

kill -l 列出所有可用信号，linux上有64个，mac上面只有31个

kill命令默认发送SIGTERM信号。
可以使用-s指定发送的信号。
kill -s SIGNAL PID  

SIGHUP  1: 对控制进程或终端的结束进行挂起检测
SIGINT  2: ctrl+c发送的信号
SIGKILL 9: 强行杀死进程，不能被捕获，也不能被忽略，必须死
SIGTERM 15:默认用于终止进程
SIGTSTP 20:ctrl+z发送的信号,将任务放到后台。

kill -9 process_id

# killall使用进程名作为参数
killall -s SIGNAL process_name
killall -9 process_name
killall -u root process_name

# pkill使用进程名作为参数，并且只支持信号编号，不支持信号名
pkill process_name
pkill -s SIGNAL process_name

# 捕获信号
trap 'function_name' SIGNAL_LIST
```

## uname
```
uname -n //hostname
uname -a //kernel version,architecture
uname -r //kernel version
uname -m //architecture
```

## cron
```
6个字段是： 分钟 小时 天 月 星期几 命令

02 * * * * /home/test.sh         // 在每天每小时到第2分钟执行
00 5,6,7 * * * /home/test.sh     // 在每天到5 6 7小时执行
00 */2 * * 0 /home/test.sh       // 在周日，每隔2小时执行一次
00 02 * * * /sbin/shutdown -h    // 每天凌晨2点关闭计算机

crontab -e可用编辑cron表

crontab<<EOF
02 * * * * /hoem/test.sh
EOF
```
Keep Fight!
{:.info}
