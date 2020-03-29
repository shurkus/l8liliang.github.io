---
layout: article
tags: Shell
title: Linux Shell脚本攻略 -- 以文件之名
mathjax: true
key: Shell
---


## dd
```
dd if=/dev/zero of=junk.data bs=1M count=1

1B = C
2B = w
512b = B
1024B = K
1024KB = M
1024MB = G

# 利用dd命令可以测试内存操作速度
dd if=/dev/zero of=/dev/null bs=1G count=10
```

## comm命令作用于两个排序过的文件，显示两个文件的差异
```
# A.txt
  apple
  orange
  gold
  silver
  steel
  iron
# B.txt
  orange
  gold
  cookies
  carrot

$comm A.txt B.txt
apple
       carrot
       cookies
               gold
iron
               orange
silver
steel
//输出的第一列包含只在A中出现的行，第二列包含只只B中出现的行，第三列包含A和B共有的行。

$ comm A.txt B.txt -1 -2 //-1表示删除第一列，-2表示删除第二列，所以这个命令只会打印共有的行
```

## 查找并删除重复的文件(下面的脚本有一个奇怪问题，没找到原因）
```
#!/bin/bash
ls -l | awk 'BEGIN{
        getline; getline; # getline读取一行后，该行被保存在$0中，行中的每一列保存在$1 $2 ... $n中
        name1=$NF; size1=$5;
}
{
        name2=$NF; size2=$5;
        print name1,size1,name2,size2;
        if ( size1==size2 )
        {
                "md5 "name1 | getline; print "xxx"$0; csum1=$NF;
                "md5 "name2 | getline; print "yyy"$0; csum2=$NF;
                print csum1,csum2;
                if ( csum1==csum2 )
                {
                        print name1; print name2;
                }
        }
        size1=size2; name1=name2;
}'
#}' | sort -u > duplicate_files

cat duplicate_files
exit 0

cat duplicate_files | xargs -I {} md5 {} | tr -d "()" | awk '{print $2" "$4;}' | \
        sort | uniq -f 1 | awk '{print $1}' | \
        sort -u > unique_files

comm -3 duplicate_files unique_files | tee /dev/stderr | xargs rm
```

## 统计文件信息
```
#!/bin/bash

if [ $# -ne 1 ];then
	echo "Usabe is $0 basepath";
	exit
fi
path=$1

declare -A statarray;

while read line;
do
	ftype=`file -b "$line" | cut -d, -f 1`
	let statarray["$ftype"]++;
done < <(find $path -t f -print)

for ftype in ${!statarray[@]};
do
	echo $ftype: ${statarray["$ftype"]}
done

# 第一个<用于输入重定向，第二个<用于将子进程的输出转换为文件名。 
# <(find $path -t f -print)等同于文件名，它用子进程输出代替文件名。
# Bash3开始，有一个新操作符<<<，可以让我们将字符串作为输入文件 done <<< "`find $path -t f -print`"
```

## 环回文件
```
//文件也可以作为文件系统挂载。这种存在于文件中的文件系统称为环回文件。可用于测试、文件系统定制、加密盘。

dd if=/dev/zero of=loopbackfile.img bs=1G count=1 // create 
mkfs.ex4t loopbackfile.img  // format
file loopbackfile.img
mkdir /mnt/loopback
mount -o loop loopbackfile.img /mnt/loopback // mount

#可以使用下面的命令指定环回设备，否则会自动创建一个环回设备
losetup /dev/loop1 loopbackfile.img
mount /dev/loop1 /mnt/loopback
```
设备上必须有文件系统才能够挂载。可以使用mkfs.ext4命令在文件中创建ext4文件系统。
当mount知道它使用的是环回文件是，会自动在dev目录创建一个环回文件的设备并将其挂载.
{:.info}


## patch
```
diff -u version1.txt version2.txt > version.patch
patch -p1 version1.txt < version.patch //执行这条命令把补丁打进verison1；再次执行这条命令会恢复version1的内容
```

## head,tail
```
head -n 4 file  //打印开头4行
head -n -4 file //打印除了最后4行的其他内容

tail -n 5 file  //打印结尾5行
tail -n +5 file //打印除了前 5-1=4 行的其他数据,就是说从第5行开始打印

tail -f file --pid $PID  //当进程结束时，tail也跟着结束
```
