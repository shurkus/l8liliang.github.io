---
layout: article
tags: Python
title: 控制python函数运行时间
mathjax: true
key: Linux
---

## yield
```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
def fab(max): 
    n, a, b = 0, 0, 1 
    while n < max: 
        yield b      # 使用 yield
        # print b 
        a, b = b, a + b 
        n = n + 1
 
for n in fab(5): 
    print n

简单地讲，yield 的作用就是把一个函数变成一个 generator，带有 yield 的函数不再是一个普通函数，Python 解释器会将其视为一个 generator，
调用 fab(5) 不会执行 fab 函数，而是返回一个 iterable 对象！在 for 循环执行时，每次循环都会执行 fab 函数内部的代码，执行到 yield b 时，
fab 函数就返回一个迭代值，下次迭代时，代码从 yield b 的下一条语句继续执行，而函数的本地变量看起来和上次中断执行前是完全一样的，
于是函数继续执行，直到再次遇到 yield。

我们可以得出以下结论：
一个带有 yield 的函数就是一个 generator，它和普通函数不同，生成一个 generator 看起来像函数调用，但不会执行任何函数代码，
直到对其调用 next()（在 for 循环中会自动调用 next()）才开始执行。
虽然执行流程仍按函数的流程执行，但每执行到一个 yield 语句就会中断，并返回一个迭代值，下次执行时从 yield 的下一个语句继续执行。
看起来就好像一个函数在正常执行的过程中被 yield 中断了数次，每次中断都会通过 yield 返回当前的迭代值。

yield 的好处是显而易见的，把一个函数改写为一个 generator 就获得了迭代能力，比起用类的实例保存状态来计算下一个 next() 的值，不仅代码简洁，而且执行流程异常清晰。
```
[yield](https://www.runoob.com/w3cnote/python-yield-used-analysis.html)
{:.info}

## contextmanager
```
from contextlib import contextmanager

@contextmanager
def file_open(path):
    try:
        f_obj = open(path,"w")
        yield f_obj
    except OSError:
        print("We had an error!")
    finally:
        print("Closing file")
        f_obj.close()

if __name__ == "__main__":
    with file_open("test/test.txt") as fobj:
        fobj.write("Testing context managers")

在这里，我们从contextlib模块中引入contextmanager，然后装饰我们所定义的file_open函数。
这就允许我们使用Python的with语句来调用file_open函数。
在函数中，我们打开文件，然后通过yield，将其传递出去，最终主调函数可以使用它。

一旦with语句结束，控制就会返回给file_open函数，它继续执行yield语句后面的代码。这个最终会执行finally语句--关闭文件。
如果我们在打开文件时遇到了OSError错误，它就会被捕获，最终finally语句依然会关闭文件句柄。
```
[contextmanager](https://www.cnblogs.com/zhbzz2007/p/6158125.html)
{:.info}

## timeout
```
from __future__ import with_statement # Required in 2.5
import signal
from contextlib import contextmanager

class TimeoutException(Exception): pass

@contextmanager
def time_limit(seconds):
    def signal_handler(signum, frame):
        raise TimeoutException, "Timed out!"
    signal.signal(signal.SIGALRM, signal_handler)
    signal.alarm(seconds)
    try:
        yield    //返回一个空对象
    finally:
        signal.alarm(0)


try:
    with time_limit(10):
        long_function_call()
except TimeoutException, msg:
    print "Timed out!"
```

