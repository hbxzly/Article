> 简单的基于Python2.7版本的多进程下开发多线程的示例

> 可以使得程序执行效率至少提升10倍

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
"""
    @Time    : 2018/10/24
    @Author  : LiuXueWen
    @Site    : 
    @File    : transfer.py
    @Software: PyCharm
    @Description: 
"""

import os
import traceback
import threading
from multiprocessing import Pool
from multiprocessing.dummy import Pool as ThreadPool


# 兼容python2.7上多线程的bug，不加上下面的反代理程序不能正常执行
def proxy(cls_instance, i):
    return cls_instance.multiprocess_thread(i)
def proxy2(cls_instance, i):
    return cls_instance.file_operation(i)

class file2transfer():
    # 多进程执行程序
    def multiprocessingTransferFiles(self):
        try:
            # 创建进程池
            p = Pool()
            //参数末尾必须加上逗号
            p.apply_async(proxy, args=(self, self.root_path,))
            p.close()
            p.join()
        except Exception as e:
            print(e)

    # 每个进程下的多线程执行，线程数等于当前机器的核数
    def multiprocess_thread(self, root_path):
        try:
            # 创建线程锁
            lock = threading.RLock()
            lock.acquire()
            # 获取每个文件
            for pfile in os.listdir(root_path):
                # 获取文件的完整路径
                full_file_path = os.path.join(root_path, pfile)
                # 多线程读写文件
                p = ThreadPool()
                # 执行线程
                p.apply_async(proxy2, args=(self, full_file_path,))
                p.close()
                p.join()
        except Exception as e:
            print(e)
        finally:
            # 释放线程锁
            lock.release()

    # 对每个文件夹下的每个文件进行操作
    def file_operation(self, full_file_path):
        try:
            // TODO 真正需要单独执行的操作
            pass
        except Exception as e:
            print(e)
```
