---
title: common file operation in python
author: wingstone
date: 2022-06-02
categories:
- python
tags:
- code snippet
metaAlignment: center
coverMeta: out
---

由于python处理文件的方便性，记录一些在python中常用的文件操作；
<!--more-->

1. 文件遍历操作

```python
import os
for root, dirs, files in os.walk("."):

    # root 表示当前正在访问的文件夹路径
    # dirs 表示该文件夹下的子目录名list
    # files 表示该文件夹下的文件list

    # 遍历文件
    for f in files:
        print(os.path.join(root, f))

    # 遍历所有的文件夹
    for d in dirs:
        print(os.path.join(root, d))
```

2. 递归删除空文件夹

```python
import os
for root, dirs, files in os.walk(".", topdown=False):
    if not files and not dirs:
        os.rmdir(root)
```

3. 递归删除后缀名文件

```python
import os
for root, dirs, files in os.walk("."):
    for f in files:
        if f.endswith(".txt"):
            os.remove(os.path.join(root, f))
```

4. 递归批处理文件

```python
from os import *
from io import *

for root, dirs, files in walk("."):
    for name in files:
        headerPath = path.join(root, name)
        if name.endswith(".txt"):

            # open file to read alllines
            fr = open(headerPath, 'r')
            lines = fr.readlines()
            fr.close()

            # open file to process write line
            fp = open(headerPath, 'w')
            for line in lines:
                line = line.replace('*','-')
                fp.write(line)
            fp.close()
```