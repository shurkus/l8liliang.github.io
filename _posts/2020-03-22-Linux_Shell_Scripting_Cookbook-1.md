---
layout: article
tags: Shell
title: Linux Shell脚本攻略 -- 小试牛刀
mathjax: true
key: Shell
---

## echo
```
# 不换行
bogon:_posts liliang$ echo -n hello
hellobogon:_posts liliang$ 

# 支持转义字符
bogon:_posts liliang$ 
bogon:_posts liliang$ echo -e "1\t2\t"
1	2

# 打印彩色输出
bogon:_posts liliang$ echo -e "\e[1;31m This is a red text \e[0m"
\e[1;31m This is a red text \e[0m
#\e[1;31是代表红色的转义字符，\e[0m代表重置。
#对于文本颜色：重置=0，黑色=30，红色=31，绿色=32，黄色=33，蓝色=34，洋红=35，青色=36，白色=37
#对于背景颜色：重置=0，黑色=40，红色=41，绿色=42，黄色=43，蓝色=44，洋红=45，青色=46，白色=47
```

## 环境变量
```
# 环境变量是从父进程继承过来的变量，export命令声明将由子进程继承的变量，一个变量被export后，当前shell脚本所执行的任何应用程序都会获得这个变量。
export PATH="$PATH:/home/user/bin"
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/local/lib"

# 检查是否为超级用户
if [ $UID -ne 0 ];then
	echo "Not root user."
else
	echo "Root user"
fi

# PS1是Bash提示符
bogon:_posts liliang$ cat /etc/bashrc | grep PS1
PS1='\h:\W \u\$ '
\u=user \h=hostname -w=pwd

# 添加环境变量包裹函数
prepend() { [ -d "$2" ] && eval $1=\"$2\$\{$1:+':'\$$1\}\" && export $1; }
prepend PATH /opt/myabb/bin

# show process ENVs
cat /proc/$PID/environ

# 几个常用变量
bogon:_posts liliang$ echo $SHELL
/bin/bash
bogon:l8liliang.github.io liliang$ echo $PWD
/Users/liliang/Leon/Git/l8liliang.github.io
bogon:l8liliang.github.io liliang$ echo $UID
501
```

## source与./
```
source : 是在当前进程执行脚本
./     : 是在子进程执行脚本

执行source a.sh之后，a.sh里面定义的所有全局变量都能在当前shell或脚本访问到
执行./a.sh之后，a.sh里面定义的所有全局变量不能在当前shell或脚本中访问到

另外要注意的是，.命令(注意这里说的不是./，而是.)等同于source，比如
source ./a.sh == . ./a.sh
```

## 数学运算
```
# let
# let命令可以直接执行基本的算术操作，并且变量名之前不需要加$
let resutl=no1+no2
let no1++
let no2--
let no1+=6

# []
result=$[ no1 + no2 ]

# (())
result=$(( no1 + 50 ))

# expr
result=$(expr $no1 + 5)

# bc
echo "scale=2;22/7" | bc
echo "obase=10;ibase=2;$no" | bc
echo "sqrt(100)" | bc
echo "10^10" | bc
```

## 重定向
```
"cmd > alloutput.txt 2>&1" == "cmd &> alloutput.txt"

# tee把输出重定向到文件并且提供一份副本给后续的管道
cat a.txt | tee -a out.txt | cat -n

# 使用'-'作为tee的文件名参数可以输出两行，也就是说'-'代表标准输出文件
echo who is this | tee -

# 自定义文件描述符
exec 3<input.txt
cat<&3
```

## 数组
```
array_var=(t1 t2 t3)
array_var[0]="t1"
array_var[1]="t2"
echo ${array_var[0]}
echo ${array_var[*]} || echo ${array_var[@]}
echo ${#array_var[*]}

# 关联数组
declare -A ass_array
ass_array=([apple]="100 dollars" [orange]="150 dollars" [waterpolen]="200 dollars")
ass_array[apple]="100 dollars"

# index
echo ${!array_var[*]}
echo ${!ass_array[@]}
```

## \command
```
在命令之前加上\可以忽略别名
```

## tput
```
#获取终端的行数和列数
tput cols
tput lines

# print name
tput longname

# set background color
tput setb 0-7

# set frontground color
tput setf 0-7

# set text bold
tput bold
```

## date & time
```
#date的精度是秒，time更精确
time commandOrScriptName
```

## debug
```
set -x   //在执行时显示参数和命令
set -v   //当命令进行读取时，显示输入

#!/bin/bash
function DEBUG()
{
	[ "$_DEBUG" == "no" ] && $@ || :
}
for i in {1..10}
do
	DEBUG echo "I is $i."
done
#在Bash中':'告诉shell不要进行任何操作，总返回0
```

## 导出函数
```
# 和环境变量一样，函数也可以导出
function getIP() { ip addr show $1 | grep 'inet '; }
echo "getIP eth0" > test.sh
export -f getIP
sh test.sh
```

## 子shell
```
#子shell本身就是一个独立的进程。可以使用（）操作符创建一个子shell
$>pwd
/
$>(cd /bin;ls)
awk bash cat...
$>pwd
/
当命令子子shell中执行时，不会对当前shell造成影响

# 通过引用子shell的方式保留空格和换行符
# 假设我们使用子shell或反引用的方式将命令的输出保存到变量中，为了保留输出的空格和换行符，必须使用双引号
$ cat test.txt
1
2
3

$ out=$(cat test.txt)
echo $out
1 2 3

$ out="$(cat test.txt)"
echo $out
1
2
3
```

## IFS（内部字段分隔符 Internal Field Separator）
```
#IFS是一个环境变量,默认值是空白符（空格 制表符 换行）
oldIFS=$IFS
IFS=,
for item in 1,2,3
do
	echo $item
done
IFS=$oldIFS
```

# {..}
```
echo {1..50}         
echo {a..z} {A..Z}
```

## bash配置文件
```
# 分三类：登录时执行的、启动交互式shell时执行的、调用shell处理脚本文件时执行的

# 用户登陆时调用
/etc/profile,$HOME/.profile,$HOME/.bash_login,$HOME/.bash_profile 
#如果是通过图形界面登录的，不调用 /etc/profile,$HOME/.profile,$HOME/.bash_profile,因为图形窗口管理器不启动shell。
#当你打开终端窗口时才会创建shell，但这个shell也不是登录shell

# 交互式shell（X11终端会话）或ssh执行单条命令（ssh 192.168.1.1 ls /tmp)是，会执行：
/etc/bashrc,$HOME/.bashrc

# 执行某个脚本时，不会执行任何配置文件，除非你指定了环境变量BASH_ENV，它才会执行你指定的BASH_ENV文件
export BASH_ENV=~/.bashrc

# 导出变量和函数会传递到子shell，但是别名不会，需要把BASH_EVN设置为.bashrc or .profile，然后在其中定义别名。

```
Keep Fight!
{:.info}

