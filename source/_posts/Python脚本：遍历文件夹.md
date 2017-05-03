---
title: Python脚本：遍历文件夹
date: 2017-04-27 23:16:35
tags: [Python]
---
<Excerpt in index | 首页摘要>
如题，实现文件夹的先序遍历和层次遍历（多叉树）。<!-- more -->
<The rest of contents | 余下全文>
通过命令：`python xx.py /xx/xx` 运行。

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
import sys
import os
import time


# 先序遍历
def preorder_traversal(path, recursion):
    if not os.path.isdir(path) and not os.path.isfile(path):  # 如果既不是文件夹也不是文件，直接返回
        return

    if os.path.isfile(path):
        return
    upper_dir = path
    for fileOrDir in os.listdir(path):
        abs_path = os.path.join(upper_dir, fileOrDir)
        if os.path.isfile(abs_path):  # 打印文件
            print fileOrDir
        elif recursion and os.path.isdir(abs_path):
            print abs_path  # 打印子文件夹
            preorder_traversal(abs_path, recursion)


# 层次遍历
def level_traversal(path):
    if not os.path.isdir(path) and not os.path.isfile(path):  # 如果既不是文件夹也不是文件，直接返回
        return
    level = 1
    for root, sub_dirs, files in os.walk(path):
        print ("============ The Level is %d ============" % level)
        level += 1
        for file_path in files:  # 打印文件
            print file_path
        for sub in sub_dirs:  # 打印子文件夹
            print os.path.join(root, sub)


pyName = sys.argv[0]
path = sys.argv[1]
print ("The running python script is : [%s] and the first argv is : [%s]" % (pyName, path))
# path = 'D:\\Work\\trytry\\github'
preorder_traversal_start_time = time.time()
preorder_traversal(path, True)
print (
    "============ preorderTraversal cost Time : %s seconds============" % (time.time() - preorder_traversal_start_time))

level_traversal_start_time = time.time()
level_traversal(path)
print ("============ levelTraversal cost Time : %s seconds============" % (time.time() - level_traversal_start_time))
```
