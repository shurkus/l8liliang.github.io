---
layout: article
tags: Linux
title: shell [ 和 ]] 的区别
mathjax: true
key: Linux
---

[soruce](https://blog.csdn.net/anqixiang/article/details/111598067)
{:.info} 


## shell [ 和 ]] 的区别
```
区别一
[ ]是符合POSIX标准的测试语句，兼容性更强，几乎可以运行在所有的Shell解释器中
[[ ]]仅可运行在特定的几个Shell解释器中(如Bash等)

区别二：<和>在[[ ]]中用作排序，而[ ]不支持

区别三：[ ]中使用-a和-o表示逻辑与和逻辑或，[[ ]]使用&&和||来表示
[[ ]]不支持-a

区别四：在[ ]中==是字符匹配，在[[ ]]中是模式匹配
[[ "$name" == c* ]] && echo Y || echo N
# name=ccc
# [[ "$name" == c* ]] && echo Y || echo N
Y
# [ "$name" == c* ] && echo Y || echo N
N

区别五：[ ]不支持正则匹配，[[ ]]支持用=~进行正则匹配
[[ "$name" =~ "c" ]] && echo Y || echo N

区别六：[ ]仅在部分Shell中支持用()进行分组，[[ ]]均支持
[[ 1 == 1 && (2 == 2 || 2 == 3) ]] && echo Y || echo N

区别七：[ ]中如果变量没有定义，那么需用双引号引起来，[[ ]]中不需要
```
