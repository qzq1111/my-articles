# Python多进程读取大文件Las(点云)文件并输出XYZ

tips:
- 本文代码基于python3编写
- 本文接上一篇[《Python解析Las(点云)格式文件》](https://github.com/qzq1111/my-articles/blob/master/Python/Python%E8%A7%A3%E6%9E%90Las(%E7%82%B9%E4%BA%91)%E6%A0%BC%E5%BC%8F%E6%96%87%E4%BB%B6.md)，在上一篇的基础上优化大文件读取和输出。
- [代码仓库](https://github.com/qzq1111/python-summary)

## Python多进程读取文件

利用多进程的并行的读取文件，可以加快读取文件的速度。废话不多说，直接上代码。

- 创建测试文件`test_multiprogress.py`

    ```python
    # coding:utf-8
    import math
    import multiprocessing
    import time
    import mmap
    import os


    # ----------------生成测试文件---------------#
    def build_txt(file, number):
        """
        模拟二进制文件，生成测试数据文件
        :return:
        """
       
        data = ["1"] * number
        with open(file, 'w') as f:
            f.writelines(data)


    # ----------------顺序读取文件---------------#
    def next_read(file, number):
        """
        顺序读取文件
        :param file:
        :return:
        """
        start_time = time.time()
        print("开始顺序读取数据")
        i = 0
        with open(file, 'r') as f:
            while i < number:
                f.read()
                i += 1
        end_time = time.time()
        print(f"读取完成耗时：{end_time - start_time}")


    # ----------------多进程读取文件---------------#
    def read(file_name, start, end):
        i = start
        with open(file_name, 'r') as f:
            f.seek(start)
            while i < end:
                f.read()
                i += 1


    def get_interval_data(number, split_number=10000):
        """
        一个进程一次性处理多少点
        :return:
        """
        num = math.ceil(number / split_number)
        data = []
        for item in range(num):
            t1 = item * split_number
            t2 = (item + 1) * split_number
            if t2 > number:
                data.append((t1, number))
                break
            data.append((t1, t2))
        return data


    def pro_read(file, number):
        """
        多进程读取文件
        """

        pool = multiprocessing.Pool(3)

        print("开始多进程读取数据")
        start_time = time.time()
        for item in get_interval_data(number):
            pool.apply_async(read, args=(file, item[0], item[1]))

        pool.close()
        pool.join()
        end_time = time.time()
        print(f"读取完成耗时：{end_time - start_time}")

    # ----------------测试---------------#
    if __name__ == '__main__':
        file = "test.txt"
        number = 200000
        build_txt(file, number)
        next_read(file, number)
        pro_read(file, number)
    ```
- 测试结果。
  ```
  开始顺序读取数据
  读取完成耗时：2.4560372829437256
  开始多进程读取数据
  读取完成耗时：1.6330537796020508
  ```
  **备注**：该测试可能由于电脑不同，结果也不一样。进程池的大小影响最后的测试结果，但不是进程池越大越好，可能适得其反。最好根据实际情况选择进程池大小。


## 内存映射技术

[内存映射](https://baike.baidu.com/item/%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84/4521338)，内存映射文件，是由一个文件到一块内存的映射。内存映射文件与虚拟内存有些类似，通过内存映射文件可以保留一个地址空间的区域，同时将物理存储器提交给此区域，内存文件映射的物理存储器来自一个已经存在于磁盘上的文件，而且在对该文件进行操作之前必须首先对文件进行映射。使用内存映射文件处理存储于磁盘上的文件时，将不必再对文件执行I/O操作，使得内存映射文件在处理大数据量的文件时能起到相当重要的作用。

在博客文章[《mmap内存文件映射》](https://www.cnblogs.com/gdk-0078/p/5165242.html)一文中，清晰的解释了两种不同方式读取文件的区别。
- 传统文件访问

    >访问文件的传统方法使用open打开他们，如果有多个进程访问一个文件，则每一个进程在再记得地址空间都包含有该文件的副本，这不必要地浪费了存储空间。下面说明了两个进程同时读一个文件的同一页的情形，系统要将该页从磁盘读到高速缓冲区中，每个进程再执行一个内存期内的复制操作将数据从高速缓冲区读到自己的地址空间。

- 共享内存映射

    > 进程A和进程B都将该页映射到自己的地址空间，当进程A第一次访问该页中的数据时，它生成一个缺页终端，内核此时读入这一页到内存并更新页表使之指向它，以后，当进程B访问同一页面而出现缺页中断时，该页已经在内存，内核只需要将进程B的页表登记项指向次页即可。

## Python mmap包的使用

- 在`test_multiprogress.py` 文件中添加mmap代码读取文件。
  
    ```python
    # ----------------内存映射方式多进程读取文件---------------#
    def open_file(file):
        """
        内存映射文件
        """
        fd = os.open(file, os.O_RDONLY)
        f = mmap.mmap(fd, 0, access=mmap.ACCESS_READ)
        return f
    
    
    def mmap_read(file, start, end):
        i = start
        f = open_file(file)
        f.seek(start)
        while i < end:
            f.read()
            i += 1
        f.close()
    
    
    def pro_mmap_read(file, number):
        """
        使用内存映射多进程读取文件
        """
    
        pool = multiprocessing.Pool(multiprocessing.cpu_count())
    
        print("开始内存映射多进程读取数据")
        start_time = time.time()
        for item in get_interval_data(number):
            pool.apply_async(mmap_read, args=(file, item[0], item[1]))
    
        pool.close()
        pool.join()
        end_time = time.time()
        print(f"读取完成耗时：{end_time - start_time}")
    
    
    # ----------------测试---------------#
    if __name__ == '__main__':
        file = "test.txt"
        number = 200000
        # 构建测试数据
        build_txt(file, number)
        # 顺序对齐
        next_read(file, number)
        # 多进程读取
        pro_read(file, number)
        # 内存映射多进程读取
        pro_mmap_read(file, number)

    ```
- 测试结果。
  ```
  开始顺序读取数据
  读取完成耗时：2.485050916671753
  开始多进程读取数据
  读取完成耗时：1.6419978141784668
  开始内存映射多进程读取数据
  读取完成耗时：0.20999979972839355
  ```
  **备注**：结果显而易见，内存映射的方式有助于大文件的读取。

## mmap多进程读Las文件并输出XYZ

代码中部分来自上篇的代码，[点击此处](https://github.com/qzq1111/python-summary/tree/master/las)查看完成代码

```python 
# coding:utf-8
import math
import mmap
import multiprocessing
import os
import struct
import time

from las.head import get_header
from las.point import Point


def get_version(f):
    """
    获取版本号
    :param f:
    :return:
    """
    f.read(24)
    version_major, version_minor = struct.unpack('2B', f.read(2))
    print(f"las版本:{version_major}.{version_minor}")
    return version_major, version_minor


def open_file(file_name):
    """
    将文件映射到内存
    :param file_name:
    :return:
    """
    fd = os.open(file_name, os.O_RDONLY)
    f = mmap.mmap(fd, 0, access=mmap.ACCESS_READ)
    return f


def get_interval_data(number_point, split_number=1000):
    """
    一个进程一次性处理多少点
    :param number_point: 总数据量
    :param split_number: 每段数据量
    :return:
    """
    num = math.ceil(number_point / split_number)

    data = []

    for item in range(num):
        t1 = item * split_number
        t2 = (item + 1) * split_number
        if t2 > number_point:
            data.append((t1, number_point))
            break
        data.append((t1, t2))
    return data


def read_point(file_name, offset_data, interval, point_format,
               x_s_f, y_s_f, z_s_f, x_o, y_o, z_o, ):
    """
    读取点数据
    :param file_name: 文件名
    :param offset_data: 文件开始位置
    :param interval: 点间隔
    :param point_format: 点数据格式
    :param x_s_f: x刻度因子
    :param y_s_f: y刻度因子
    :param z_s_f: z刻度因子
    :param x_o: x偏移量
    :param y_o: y偏移量
    :param z_o: z偏移量
    :return:
    """
    lower_interval, upper_interval = interval

    point = Point(x_s_f,
                  y_s_f,
                  z_s_f,
                  x_o,
                  y_o,
                  z_o,
                  point_format
                  )

    num = upper_interval - lower_interval
    lower_interval = offset_data + lower_interval * (point.offset_bytes + 12)
    f = open_file(file_name)
    print("开始测试读取点数据")
    start = time.time()
    data = point.read_point(f, lower_interval, num)
    end = time.time()
    print(f"完成消耗时间:{end - start}")
    f.close()
    return data


def write_file(data):
    """
    输出到文件
    :param data:
    :return:
    """
    with open("point1.txt", 'a+') as f:
        f.writelines(data)


if __name__ == '__main__':
    file_name = 'test.las'
    f = open_file(file_name)
    version = get_version(f)
    header = get_header(f, version)
    f.close()

    print(header.__dict__)

    interval_data = get_interval_data(header.number_of_point_records, 10000)
    pool = multiprocessing.Pool(multiprocessing.cpu_count())

    print("开始测试读取点数据")
    write_start = time.time()
    for item in interval_data:
        pool.apply_async(read_point,
                         args=(file_name,
                               header.offset_to_point_data,
                               item,
                               header.point_data_record_format,
                               header.x_scale_factor,
                               header.y_scale_factor,
                               header.z_scale_factor,
                               header.x_offset,
                               header.y_offset,
                               header.z_offset,
                               ),
                         callback=write_file
                         )

    pool.close()
    pool.join()
    write_end = time.time()
    print(f"完成消耗时间:{write_end - write_start}")

```


**备注**：多进程情况下写同一文件，可能会出现数据丢失的情况。为保证数据的完整性，使用`callback`。

## 总结

- mmap功能十分强大，有利于大文件的读取。
- 大致实现了多进程下读取las文件及输出XYZ。
