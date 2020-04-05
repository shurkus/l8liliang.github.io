---
layout: article
tags: Shell
title: Linux Shell脚本攻略 -- 归档
mathjax: true
key: Shell
---

## tar 

```
# 归档
tar -cf output.tar [SOURCES]

# -t列出归档文件中包含的文件
tar -tf archive.tar

# -r追加
tar -rvf original.tar new_file

# 提取
tar -xf archive.tar
tar -xf archive.tar -C /path/to/extraction_directory

# 指定stdout为输出文件
tar -cvf - files | ssh user@hostname "tar -xv -C /path/to/extraction_directory"

# 合并两个归档文件
tar -Af file1.tar file2.tar

# 当文件比归档文件中的同名文件更新时，才追加进去
tar -uf archive.tar filea

# 把归档文件与当前文件系统中的内容进行比较
tar -df archive.tar

# 从归档中删除文件
tar -f archive.tar --delete file1 file2 ..
tar --delete --file archive.tar [FILE LIST]

# compress
-j     -->  bunzip2
-z     -->  gzip
--lzma -->  lzma

//-a基于输出文件扩展名选择压缩格式
tar -acvf archive.tar.gz filea fileb filec // will select gzip

# 排除部分文件
tar -cf arch.tar * --exclude "*.txt"
tar -cf arch.tar * -X list_file    // 将要排除的文件写进文件中

# 打印归档总字节数
tar -cf arc.tar * --exclude "*.txt" --totals
```

## cpio
```
cpio通过stdin获取输入文件名并将归档文件写入stdout。
有三种操作模式：copy-out， copy-in， copy-pass

ls file* | cpio -ov > archive.cpio  // -o进入copy-out操作模式，就是打包的意思
cpio -it < archive.cpio             // -i进入copy-in操作模式，t表示列出包里面的文件名
cpio -id < archive.cpio             // -d在需要的时候创建目录
```
[一篇关于cpio的文章](https://www.jianshu.com/p/d222e00faae1)
{:.info}

## gzip
```
gzip filename                 // 压缩 --fast --best指定压缩级别
gunzip filename.gz            // 解压缩
gzip -l test.txt.gz           // 列出压缩文件到属性信息
cat file | gzip -c > file.gz  // -c表示把输出写到stdout
ls * | cpio -o | gzip -c > cpiooutput.gz
zcat cpiooutput.gz | cpio -it  // zcat在不解压缩到情况下显示文件内容

tar -zcvf archive.tar.gz [FILES]

tar -cvf archive.tar [FILES]
gzip archive.tar

tar -zxvf archive.tar.gz
tar -zxvf archive.tar.gz

#压缩率
1级最低
9级最高
默认6
gzip -5 test.img
```

## bzip2
```
bzip2比gzip压缩效率更高，但是更慢

bzip2 filename           // compress
bunzip2 filename.bz2     // uncompress
tar -jcvf archive.tar.bz2 [Files]
tar -jxvf archive.tar.bz2
```

## lzma
```
lzma的压缩率优于gip和bzip2
lzma filename            // compress
unlzma filename.lzma     // uncompress

tar -cvf --lzma archive.tar.lzma [Files]
tar -xvf --lzma archive.tar.lzma
```

## zip
```
zip多平台通用(Linux，Mac，Windows)，可以跨平台分发数据

zip archive_name.zip file1 file2 ...
zip -r archive.zip folder1 folder2      // -r recursive

unzip archive.zip    //提取之后并不会删除archive.zip

zip file.zip -u newfile //更新归档

zip -d file.zip a.txt //从归档中删除文件

unzip -l archive.zip  //列出文件名
```

## pbzip2
```
多线程bzip2，更快

pbzip2 myfile.tar
tar -cf file.tar.bz2 --use-compress-prog=pbzip2 dir
tar -c dir/ | pbzip2 -c > myfile.tar.bz2

pbzip2 -d myfile.tar.bz2 // -d解压缩
pbzip2 -dc myfile.tar.bz2 | tar -x

# specify process number
pbzip2 -p4 myfile.tar

# 也可以指定压缩比 -1 ~ -9
```

## rsync
```
rsync在最小化数据传输的同时，同步不通位置上的文件和目录。相较于cp命令，rsync的优势在于比较文件修改日期，只复制较新的文件。
另外它还支持远程数据传输和加密。

rsync -av source_path destination_path

rsync -av /home/slynux/data slynux@192.168.0.6:/home/backups/data

-a 表示进行归档操作
-v verbose
-z 可以指定在远程传输时压缩数据

rsync -av /home/test/ /home/backups  //不复制test目录本身
rsync -av /home/test /home/backups   //包括test目录本身

# --exclude --exclude-from可以指定不需要备份的文件

# --delete 在目的端删除在源端已经不存在的文件
rsync -avz SOURCE DEST --delete
```

Keep Fight!
{:.info}
