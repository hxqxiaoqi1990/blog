---
title: python2之threading模块多线程
tags: [python]
categories: linux基础
date: 2019-11-02 20:11:11
---


# 介绍
多线程类似于同时执行多个不同程序，多线程运行有如下优点：

1.使用线程可以把占据长时间的程序中的任务放到后台去处理。
2.用户界面可以更加吸引人，这样比如用户点击了一个按钮去触发某些事件的处理，可以弹出一个进度条来显示处理的进度
3.程序的运行速度可能加快
4.在一些等待的任务实现上如用户输入、文件读写和网络收发数据等，线程就比较有用了。在这种情况下我们可以释放一些珍贵的资源如内存占用等等。

# 简单的示例
## 示例一
手动创建三个线程同时执行，打印和后，等待5秒结束。

``` python
# -*- coding: UTF-8 -*-
import time
import threading

def f1(a1, a2):
    print a1 + a2
    time.sleep(5)

# 创建线程
t = threading.Thread(target=f1, args=(111, 111))  
# 开启线程
t.start()  

t = threading.Thread(target=f1, args=(222, 222))
t.start()

t = threading.Thread(target=f1, args=(333, 333))
t.start()
```

## 示例二
这段代码执行的效果是2秒内全部执行完成，而不是每执行一次函数等待两秒，打开了三个线程同时执行函数，打印三次时间。

``` python
# -*- coding: UTF-8 -*-
import threading
import time

# 创建一个函数交给多线程执行
def test(x):
    date = time.time()
    print x
    print date
    time.sleep(2)

threads = []

# target=执行的函数，args=传给函数的值，range代表打开几个线程执行，变量i传给函数test的x
for i in range(3):
    t = threading.Thread(target=test, args=(i,))
    threads.append(t)

# 打开线程活动
for thr in threads:
    thr.start()

# 等待至线程中止
for thr in threads:
    thr.join()
```