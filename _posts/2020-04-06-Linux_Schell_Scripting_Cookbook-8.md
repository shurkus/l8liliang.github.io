---
layout: article
tags: Shell
title: Linux Shell脚本攻略 -- 网络
mathjax: true
key: Shell
---

## DNS 

```
host命令可以列出某个域名的所有ip
nslookup可以指定查询的类型，可以查到DNS记录的生存时间，还可以指定使用哪个DNS服务器进行解释。
可以通过向/etc/hosts中加入条目来实现名字自解析
echo 192.168.0.2 server2 >> /etc/hosts
```

## 并行ping
```
# method 1
for ip in 192.168.0.{1..255};
do
	(
		ping $ip -c 2 &> /dev/null;
		if [ $? -eq 0 ];then
			echo $ip is alive
		fi
	)&
done
wait

# method 2: fping
-a显示出所有活动主机的ip
-u显示出所有不可达主机
-g指定从“ip地址/子网掩码“记法或者”ip地址范围记法生成一组ip

fping -a 192.168.1/24 -g
fping -a 192.168.1.1 192.168.1.2 ...
```

## ssh
```
ssh username@remote_host
ssh username@remote_host -p 422
ssh mec@192.168.0.1 'command1;command2;...'
ssh -C user@hsotname COMMANDS // -C对数据进行压缩
echo 'text' | ssh user@hostname 'echo'  //使用本地系统命令的输出作为远程系统命令的输入

# 在远程主机执行图形化命令
ssh user@host "export DISPLAY=:0; command1; command2; ..."  //这种方式强制把应用连接到远程主机的Xserver
ssh -X user@host "command1; command2; ..."    //这种方式将图形显示在本地主机上
```
[关于DISPLAY的一篇文章](http://blog.chinaunix.net/uid-23072872-id-3388906.html)
{:.info}

## ssh无密码自动登录
```
ssh采用非对称加密技术，认证密钥包含两部分：一个是公钥，一个是私钥。
公钥放到目标服务器的~/.ssh/authorized_keys中，私钥放在自己的~/.ssh目录中。

ssh-keygen -t rsa // 创建公钥~/.ssh/id_rsa.pub和私钥~/.ssh/id_rsa
ssh user@remote "cat >> ~/.ssh/authorized_keys" < ~/.ssh/id_rsa.pub  //把公钥添加到远程主机

ssh-copy-id user@remote  // 该工具专门用来把公钥添加到远程主机
```

## ssh端口转发
```
ssh -L 8000:www.kernel.org:80 user@localhost  // 将本地端口8000的流量转发到www.kernel.org:80上
ssh -L 8000:www.kernel.org:80 user@remote     // 将远程主机端口8000的流量转发到www.kernel.org:80

ssh -fL 8000:www.kernel.org:80 user@localhost -N // -f指定ssh执行命令前转入后台 -N告诉ssh无需执行命令，这样就不用总保持shell打开就可以进行端口转发了

ssh -R 8000:localhost:80 user@remote    // 将远程主机端口8000的流量转发到本地80端口
```

## FTP自动执行命令
```
lftp是更新的ftp，包含下面的命令：
cd dir ：更改远程主机目录
lcd    ：更改本地主机目录
mkdir  ：在远程主机创建目录
ls     ：列出远程主机当前目录下的文件
get fn ：下载
put fn ：上传
quit   ：退出

#!/bin/bash
HOST=example.com
USER="foo"
PASSWD="password"
ftp -u ${USER}:${PASSWD} $HOST <<EOF

binary
cd /home/foo
put testfile.jpg

quit
EOF

脚本中的EOF结构用来通过stdin向ftp发送数据。
```

## 在本地挂载点挂载远程驱动器
```
sshfs -o allow_other user@remote:/home/path /mnt/mountpoint
umount /mnt/mountpoint
```

## nc
```
nc -l 1234 > dest_file
nc HOST 1234 < source_file
```

## lsof (list open files)
```
lsof -i | egrep ":[0-9]+->" -o | egrep "[0-9]+" -o | sort | uniq //列出主机上当前开放的端口
```

## iptables
```
iptables中的第一个选项可以是-A，表明向chain中添加一条规则；
也可以是-I，表明将新规则插入到chain的开头。

-d     目的地址
-s     源地址
-j     动作
-p     协议
-dport 目的端口
-i     入接口
-o     出接口

#!/bin/bash
#name: netsharing.sh
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -A FORWARD -i $1 -o $2 \
	-s 10.99.0.0/16 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A POSTROUTING -t nat -j MASQUERADE

# firewall
iptables -A OUTPUT -d 8.8.8.8 -j DROP
iptables -A OUTPUT -p tcp -dport 21 -j DROP
iptables -I INPUT -s 1.2.3.4 -j DROP
```

Keep Fight!
{:.info}
