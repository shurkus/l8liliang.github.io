---
layout: article
tags: Shell
title: Linux Shell脚本攻略 -- 命令之乐
mathjax: true
key: Shell
---


## cat
```
cat -s  // remove null line
cat -T  // display Tab as ^I
cat -n  // display line number
```

## script,scriptreplay
```
script -t 2>timing.log -a output.session // start record
execute some commands
exit

scriptreplay timing.log output.session  // replay

```

## find
```
# find命令工作方式：沿着文件层次结构向下遍历，匹配符合条件的文件，执行相应的操作。默认操作是打印出文件和目录名，即-print选项。
# 可以通过-exec指定你要执行的命令
# -print使用换行符\n分隔输出的每个文件或目录名；-print0使用空字符\0来分隔。

# find命令可以根据通配符或者正则表达式进行搜索
find /home/slynux -name '*.txt' -print  // 在/home/slynux查找txt文件。
find . -iname "example*" -print         // 匹配时忽略名字的大小写
find . \( -name "*.txt" -o -name "*.pdf" \) -print //支持逻辑与、逻辑或
find /home/users -path '*/slynux/*' -name '*.txt' -print //通过-path选项限制匹配的路径
find . -regex '.*\.\(py\|sh\)$'  //通过正则表达式匹配所有py和sh文件

# regex
find . -iregex '.*\.\(py\|sh\)$'  //通过正则表达式匹配所有py和sh文件，或略大小写

# negative
find . ! -name "*.txt" -print

# maxdepth mindepth控制目录深度
find -L /proc -maxdepth 1 -name 'bundlemaker.def' 2>/dev/null // -L跟随符号链接
find . -mindepth 2 -name "f*" -print
# maxdepth mindepth在命令行中应该尽早出现来提高效率。

# type
find . -type d // directory
find . -type f // file
find . -type l // link
find . -type c // char device
find . -type b // block device
find . -type s // socket
find . -type p // pipe

# time
-atime:最近一次访问时间
-mtime:最近一次修改时间
-ctime:文件元数据（权限或者所有者）最后一次改变时间
find . -type f -atime -7 -print //7天内被访问的文件
find . -type f -atime 7 -print  //刚好七天前被访问的文件
find . -type f -atime +7 -print //最后一次访问时间超过7天的文件

-amin
-mmin
-cmmin
是对应的以分钟为单位的选项。

# size
find . -type f -size +2k
find . -type f -size -2k
find . -type f -size 2k

# permission
find . -type f -perm 644 -print
find . -type f ! -perm 644 -print
find . -type f -user slynux -print

# -delete
find . -type f -name "*.swap" -delete //删除文件

# -exec
find . -type f -user root -exec chown slynux {} \;
// {}代表匹配到的文件名
// 结尾的;必须转义，否则shell会认为是find结束，而非chown的结束

# 合并目录下的所有c文件，三种方式
find . -type f -name "*.c" -exec cat {} \;>all_c_files.txt
find . -type f -name "*.c" -exec cat {} >all_c_files.txt \;
find . -type f -name "*.c" -exec cat {} >all_c_files.txt +
// 没有使用追加符>>而是使用>的原因是，find命令的全部输出只有一个数据流

# 使用-prune跳过目录
find devel/source_path -name '.git' -prune -o -type f -print //skip .git directory
```
Unix默认并不保存文件创建时间。但是有一些文件系统会这么做。使用stat命令可以访问文件创建时间。
但是有些应用通过删除再创建的方式修改文件，所以文件创建时间未必准确。
{:.info}

## xargs
```
# xargs从stdin读取数据，然后把这些数据作为参数传给其他命令去执行
ls *.c | xargs grep main //在所有c文件中查找字符串main

# xagrs默认执行echo命令
cat example.txt
1 2 3 4 5 6 
7 8 9 10
11 12

cat example.txt | xargs
1 2 3 4 5 6 7 8 9 10 11 12

# -n限制每次调用命令时用到的参数个数
cat example.txt | xargs -n 3
1 2 3
4 5 6
7 8 9
10 11 12

# xagrs默认使用空白符分隔输入
# 可以使用-d指定分隔符
echo "split1Xsplit2Xsplit3" | xargs -d X

# -0选项表示用空字符分隔输入
find /smbMount -iname '*.docx' -print0 | xargs -0 grep -L image

# -I选项指定替换字符
cat args.txt | xargs -I {} ./cecho.sh -p {} -1

# 通过子shell为多组命令提供参数
cat files.txt | ( while read arg; do cat $arg; done } 等同于 cat files | xargs -I {} cat {}
// 但是可以在while循环中将cat命令替换成任意数量多命令，这样就可以对同一个参数执行多条命令。

# shell -c调用子shell来执行脚本
find . -name '*.c' | xargs -I ^ sh -c "echo -ne '\n ^: '; grep main ^"

```

## tr
```
# tr [options] set1 set2 // tr从stdin也只能从stdin接收输入，把set1中的第一个字符映射到set2中的第一个字符，依此类推，把结果写到stdout
echo "HELLO" | tr 'A-Z' 'a-z' //如果起始字符到终止字符不是有效的连续字符，那么这个集合就会被当作只有三个字符的集合。

# tr -d 删除集合中的字符
cat file.txt | tr -d 'a-c'

# 补集
echo hello 1 cahr 2 next 4 | tr -d -c '0-9\n' // -c -d同时使用的时候，只能有set1，表示删除非set1中的字符
>>124
echo hello 1 char 2 next 4 | tr -c '0-9' ' ' // set1和set2同时使用，表示把不在set1中的字符替换为set2中的字符
>>       1      2      4

# -s 删除重复出现的字符
cat multi_linx.txt | tr -s '\n'

# 妙用
cat sum.txt
1
2
3
4
5
cat sum.txt | echo $[ $(tr '\n' + ) 0 ] // 最后执行 echo $[ 1+2+3+4+5+0 ],说明了只有接受stdin的命令tr会被重复执行，echo命令只执行一次

# 字符类
alnum:字母和数字
alpha:字母
cntrl:控制字符（非打印字符）
digit:数字
graph:图形字符
lower:小写字符
print:可打印字符
punct:标点符号
space:空白字符
upper:大写字符
xdigit:十六进制字符

tr '[:lower:]' '[:upper:]'
```

## sort
```
sort file1 file2 > sorted
sort file1 file2 -o sorted
sort -n file1  //按照数字顺序排序
sort -r file1  //逆序

sort -m sorted1 sorted2 //合并两个已经排序的文件

sort -c filename; //检查是否已经排序

# -k指定排序所依据的字符。可以是单个字符，表示列号；也可以是2.3,2.4表示使用第2列第第3个字符排序
sort -nrk 1 data.txt
sort -bk 2.3,2.4 data.txt 
sort -bd unsorted.txt // -d表示以字典序排序
```

## uniq 只能作用于排序好的数据
```
# uniq -u 只显示一行
# uniq -c 统计各行出现的次数
# uniq -d 找出重复的行
# uniq -S 指定跳过前N个字符
# uniq -w 指定用于比较多最大字符数
# uniq -z 生成由0值字节终止的输出
```

## mktemp
```
filename=`mktemp`
dirname=`mktemp -d`
tmpfile=`mktemp -u` //只输出文件名，不创建文件
mktemp test.XXX //基于模版创建
```

## look
```
# look命令可以显示出特定字符串在文件中的起始行，默认搜索/usr/share/dict/words
look word
look 'Aug 30' /var/log/syslog
```

Keep Fight!
{:.info}
