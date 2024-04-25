---
layout: article
tags: Python
title: decorator
mathjax: true
key: Linux
---

[soruce](https://www.cnblogs.com/tobyqin/p/python-decorator.html)
{:.info} 

## 基本的装饰器
```
概括的讲，装饰器的作用就是为已经存在的函数或对象添加额外的功能。

def debug(func):
    def wrapper(*args, **kwargs):  # 指定宇宙无敌参数
        print "[DEBUG]: enter {}()".format(func.__name__)
        print 'Prepare and say...',
        return func(*args, **kwargs)
    return wrapper  # 返回
 
@debug
def say(something):
    print "hello {}!".format(something)


```

## 带参数的装饰器
```
def logging(level):
    def wrapper(func):
        def inner_wrapper(*args, **kwargs):
            print "[{level}]: enter function {func}()".format(
                level=level,
                func=func.__name__)
            return func(*args, **kwargs)
        return inner_wrapper
    return wrapper
 
@logging(level='INFO')
def say(something):
    print "say {}!".format(something)
 
# 如果没有使用@语法，等同于
# say = logging(level='INFO')(say)
 
@logging(level='DEBUG')
def do(something):
    print "do {}...".format(something)
 
if __name__ == '__main__':
    say('hello')
    do("my work")
```
