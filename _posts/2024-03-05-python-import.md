---
layout: article
tags: Python
title: import
mathjax: true
key: Linux
---

[soruce](https://c.biancheng.net/view/2397.html)
{:.info} 


## import 整个模块(需要使用前缀)
```
# 导入sys整个模块
import sys
# 使用sys模块名作为前缀来访问模块中的成员
print(sys.argv[0])

# 导入sys、os两个模块
import sys,os
```

## 设置别名(需要使用前缀)
```
# 导入sys整个模块，并指定别名为s
import sys as s
# 使用s模块别名作为前缀来访问模块中的成员
print(s.argv[0])

# 导入sys、os两个模块，并为sys指定别名s，为os指定别名o
import sys as s,os as o
```

## from xx import x1（无须使用任何前缀）
```
# 导入sys模块的argv成员
from sys import argv
# 使用导入成员的语法，直接使用成员名访问
print(argv[0])
```

## from xx import x1 as y（无须使用任何前缀）
```
# 导入sys模块的argv成员，并为其指定别名v
from sys import argv as v
# 使用导入成员（并指定别名）的语法，直接使用成员的别名访问
print(v[0])
```

## from xx import x1,x2（无须使用任何前缀）
```
# 导入sys模块的argv,winver成员
from sys import argv, winver
# 使用导入成员的语法，直接使用成员名访问
print(argv[0])
print(winver)
```

## from xx import x1 as a, x2 as b（无须使用任何前缀）
```
# 导入sys模块的argv,winver成员，并为其指定别名v、wv
from sys import argv as v, winver as wv
# 使用导入成员（并指定别名）的语法，直接使用成员的别名访问
print(v[0])
print(wv)
```

## from xx import * (注意多个模块中名字的冲突)
```
#导入sys 棋块内的所有成员
from sys import *
#使用导入成员的语法，直接使用成员的别名访问
print(argv[0])
print(winver)
```

## from xx.yy import zz
```
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import LabelEncoder
from sklearn.linear_model import LinearRegression
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_squared_log_error
```

