---
layout: article
tags: Shell
title: Linux Shell脚本攻略 -- 让文本飞
mathjax: true
key: Shell
---

## regexper
```
# position marker anchor 位置标记锚点

# 标志符
  A
  .
  []
  [^]

# 数量修饰符
  ?
  +
  *
  {n}
  {n,m}
  {n,}

# other
  () 将括号中的内容视为一个整体
  |
  \

# -c统计匹配的行树

# -n打印行号

# -b打印出匹配内容出现在行中的偏移

# -l打印出匹配该模式的文件名

# -L打印出不匹配该模式的文件名

# -E -v -o 

# -R is eqaul to -r

# -e指定多个模式
grep -e "patten1" -e "patten2"

# 打印匹配的数量
echo -e "1 2 3 4\nhello\n 5 6" | egrep -o "[[:digit:]]" | wc -l

# --include指定搜索特定文件
grep "main()" . -r --include *.{c,cpp}

# --exclude

# --exclude-dir

# --exclude-from File

# -Z告诉grep使用\0作为文件名终结符
grep "test" file* -lZ | xargs -0 rm

# -q silent

# -A,-B,-C

# \b在正则中表示边界，\B表示非边界，\w字母数字下划线，\W非字母数字下划线
sed 's/\b[0-9]\{3\}\b/NUMBER/' sed_data.txt 
```

## cut
```
cut -f 2,3 filename
cut -f3 --complement student_data.txt //反选 打印除了第3列之外的列
cut -f2 -d ":" filename   // -d设置分隔符

# 指定范围
cut -c2-5 filename
cut -c2- filename
cut -c-5 filename
# c --> char
# b --> byte
# f --> field
```

## sed (stream editor)
```
# -i in place，表示直接修改文件而不是只把修改后的内容打印到输出

# 替换每行中模式首次匹配的内容
sed 's/pattern/replace_string/' file

# 使用g标记执行全局替换
sed 's/pattern/replace_string/g' file

# sed命令会将s之后的字符视为命令分隔符。这允许我们更改默认分隔符
sed 's:text:replace:g'
sed 's|text|replace|g'

# 移除空行
sed '/^$/d' file

# 已匹配字符串 &
echo this is an example | sed 's/\w\+/[&]/g'
[this] [is] [an] [example]

# 字串匹配标记 \1 \2 ...
echo this is digit 7 in a number | sed 's/digit \([0-9]\)/\1/'
> this is 7 in a number
echo "seven EIGHT" | sed 's/\([a-z]\+\) \([A-Z]\+\)/\2 \1/'
> EIGHT seven

# 删除行首空格
sed 's/^[ \t]*//g'
# 删除行尾空格
sed 's/[ \t]*$//g'
# 删除所有空格
sed s/[[:space:]]//g
```

## awk
```
awk 'BEGIN { print "start" } pattern { commands } END { print "end" }' file

NR,NF,$0,$1 ...

# awk 支持printf函数，语法和c语言的同名函数一样
awk 'BEGIN { printf("%-10s%-4s\n","Name","Age");printf("%-10s%-4s\n","----","---");} { printf("%-10s%-4d\n",$1,$2) }' data.txt
Name      Age
----      ---
leon      20
jack      22
mark      18

# 传入外部参数
VAR=10000
echo | awk -v VARIABLE=$VAR '{ print VARIABLE }'

awk '{ print v1,v2 }' v1=xxx v2=xxx file

# getline 读取一行

# filter
awk 'NR < 5'
awk 'NR==1,NR==4' # between line 1 and line 4
awk '/linux/'
awk '!/linux/'  # reverse select

# awk 的关联数组更好用
arrayName[index]
arrayName[index]=value

# awk for
for(i=0;i,10;i++) { print $i; }
for(in in array) { print array[i]; }

awk 'BEGIN {FS=":"} {name[$1]=$5} END {for (i in name) {print i,name[i]}}' /etc/passwd

# awk处理字符串很方便
length(string)
index(string,search_string)
split(string,array,delimiter)
substr(string,start-position,end-position)
sub(regex,replacement_str,string)     // 将第一个匹配到内容替换
gsub(regex,replacement_str,string)    // 替换所有匹配
match(regex,string)                // 匹配返回0；否则返回1；RSTART,RLENGTH.

# 打印指定行或者指定模式之间的文本
awk 'NR==M, NR==N' filename

awk '/start_pattern/, /end_pattern/' filename
```

## paste 合并文件
```
paste file1 file2 -d ","
1,slynux
2,gnu
3,bash
4,hack
5,
```

## text slice
```
var="This is a line of text"
echo ${var/line/REPLACED}
> This is a REPLACED of text"

echo ${var:3:8} //substring
> s is a li
echo ${var:(-1)}
> t
echo ${var:(-2):2}
> xt
```

Keep Fight!
{:.info}
