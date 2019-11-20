# Python解析Las(点云)格式文件

tips:
- 本文代码基于python3编写
- [代码仓库](https://github.com/qzq1111/python-summary)

## 一、点云

- 点云：
  
  在逆向工程中通过测量仪器得到的产品外观表面的点数据集合也称之为点云，通常使用三维坐标测量机所得到的点数量比较少，点与点的间距也比较大，叫稀疏点云；而使用三维激光扫描仪或照相式扫描仪得到的点云，点数量比较大并且比较密集，叫密集点云。

- 点云格式：
  
  `*.las`、`*.pcd`、`*.txt`

## 二、Las格式
### 1. 简介
las文件是一个二进制文件，其中定义的数据类型与C语言中数据类型一致。到目前为止，las共有6版分别是：
- [Superseded ASPRS LAS 1.4 Format Specification R14 March 26 2019 (PDF)](http://www.asprs.org/wp-content/uploads/2019/03/LAS_1_4_r14.pdf)
- [Superseded ASPRS LAS 1.4 Format Specification R13 July 15 2013 (PDF)](https://www.asprs.org/wp-content/uploads/2010/12/LAS_1_4_r13.pdf)
- [Superseded ASPRS LAS 1.3 Format Specification October 24 2010 (PDF)](https://www.asprs.org/wp-content/uploads/2010/12/LAS_1_3_r11.pdf)
- [Superseded ASPRS LAS 1.2 Format Specification September 2 2008 (PDF)](https://www.asprs.org/wp-content/uploads/2010/12/asprs_las_format_v12.pdf)
- [Superseded ASPRS LAS 1.1 Format Standard May 7 2005 (PDF)](https://www.asprs.org/wp-content/uploads/2010/12/asprs_las_format_v11.pdf)
- [Superseded ASPRS LAS 1.0 Format Standard May 9 2003 (PDF)](https://www.asprs.org/wp-content/uploads/2010/12/asprs_las_format_v10.pdf)
  
### 2. Las规范数据类型
|数据类型|字节|
|---|---|
|char|1 byte|
|unsigned char|1 byte|
|short |2 bytes|
|unsigned short|2 bytes|
|long |4 bytes|
|unsigned long|4 bytes|
|double  |8 bytes|

### 3. las文件整体构成

|las1.0~las1.2|las1.3~las1.4|
|---|---|
|公共头| 公共头|
|变长记录域|变长记录域|
|点数据域|点数据域|
||扩展变长记录域|

### 4. 公共头不同版本构成 

|las1.0~las1.2|las1.3~las1.4|
|---|---|
|文件标识（“LASF”）|文件标识（“LASF”）|
|保留字节|文件ID|
|GUID数据|全球编码|
|版本信息|GUID数据|
|系统标识|版本信息|
|飞行时间|系统标识|
|头部大小|文件创建时间|
|数据偏移|头部大小|
|变长记录域数目|数据偏移|
|点数据格式、长度、数目、不同回波点数目|变长记录域数目|
|X、Y、Z刻度因子|点数据格式、长度|
|X、Y、Z偏移值|老格式点数目、不同回波点数目|
|X、Y、Z最大最小值|X、Y、Z刻度因子|
||X、Y、Z偏移值|
||X、Y、Z最大最小值|
||波形数据包记录起始位置|
||扩展变长记录起始、数目|
||点数目、不同回波点数目|

 
### 5. 点记录格式说明

在las1.0版本中定义了点数据格式0，其一共**20字节**数据，在las1.0~las1.4的版本中点数据格式1到5都是在点数据格式0基础上增添字段。具体字段说明可参见不同版本的[官方文档](https://www.asprs.org/divisions-committees/lidar-division/laser-las-file-format-exchange-activities)。

在las1.4版本中增加了点格式6，其一共**30字节**数据，在las1.4版本中点格式7到10都是在点数据格式6基础上增添字段。具体字段说明可参见不同版本的[官方文档](https://www.asprs.org/divisions-committees/lidar-division/laser-las-file-format-exchange-activities)。

**备注**：

在点格式0~5中`Return Number`、`Number of Returns (given pulse)`、`Scan Direction Flag`、`Edge of Flight Line`等字段共占1字节数据。

在点格式6~10中`Return Number`、`Number of Returns`、 `Classification Flags`、 `Scan Direction Flag`、  `Edge of Flight Line`等字段共占2字节数据。

这些字段的值都是通过**位计算**获得。

## 三、Python解析二进制

Python解析二进制文件使用内置`struct`包

1. Python数据类型与C语言数据类型对应关系，[参考链接](https://docs.python.org/3/library/struct.html#format-characters)。

   |Format|	C Type|	Python|	字节数|
   |---|---|---|---|
   |x|pad byte|no value|1|
   |c|char|	bytes of length 1|1|
   |b|signed char|integer|	1|
   |B|unsigned char|integer|1|
   |?|_Bool|bool|1|
   |h|short|integer|2|
   |H|unsigned short|integer|2|
   |i|int|integer|4|
   |I|unsigned int|integer|4|
   |l|long|integer|4|
   |L|unsigned long|integer|4|
   |q|long long|integer|8|
   |Q|unsigned long long|integer|8|
   |f|float	|float|4|
   |d|double|float|8|
   |s|char[]|bytes|1|
   |p|char[]|bytes|1|
   |P|void *|integer||
2. 简单使用`struct`包
   
   ```python
   import struct

   # 创建float型二进制数据
   a = struct.pack('f', 12.34)
   print(f"转换的二进制：{a}", )
   # 创建char[]类型二进制数据
   b = struct.pack('4s', 'test'.encode('utf-8'))
   print(f"转换的二进制：{b}", )

   # 写入到二进制文件
   f = open('test.dat', 'wb+')
   f.write(a)
   f.write(b)
   f.close()

   # 读取二进制文件
   f = open('test.dat', 'rb')
   a = f.read(4)
   print(f"读取的二进制：{a}", a)
   a = struct.unpack('f', a)
   print(f"转换为Python数据类型：{a}", )
   b = f.read(4)
   print(f"读取的二进制：{b}", )
   b = struct.unpack('4s', b)
   print(f"转换为Python数据类型：{b}", )
   f.close()
   ```
## 四、读取las文件并提取XYZ坐标值

### 1. 读取公共头
- 新建`head.py`，定义las1.0~las1.4版本不同的公共头的类,具体实现[参考此处](https://github.com/qzq1111/python-summary/blob/master/las/head.py)
  
  ```python
  class OneZeroHeader(object):
    """
    1.0版本
    """
    def __init__():
        pass
    
    def read_header(f):
        pass

  class OneOneHeader(object):
    """
    1.1版本
    """
    def __init__():
        pass
    
    def read_header(f):
        pass

  class OneTwoHeader(object):
    """
    1.2版本
    """
    def __init__():
        pass
    
    def read_header(f):
        pass

  class OneThreeHeader(object):
    """
    1.3版本
    """
    def __init__():
        pass
    
    def read_header(f):
        pass

  class OneFourHeader(object):
    """
    1.4版本
    """
    def __init__():
        pass
    
    def read_header(f):
        pass

  def get_header(f, version):
    """
    根据不同版本读取不同头
    """
    if version == (1, 0):
        new_header = OneZeroHeader()
    elif version == (1, 1):
        new_header = OneOneHeader()
    elif version == (1, 2):
        new_header = OneTwoHeader()
    elif version == (1, 3):
        new_header = OneThreeHeader()
    elif version == (1, 4):
        new_header = OneFourHeader()
    else:
        raise Exception("未找到对应文件版本")

    new_header.read_header(f)
    return new_header
  ```

- 新建`test_read_las.py`，读取公共头数据
  
  ```python

  import struct
  from las.head import get_header
  
  
  def get_version(f):
      f.read(24)
      version_major, version_minor = struct.unpack('2B', f.read(2))
      print(f"las版本:{version_major}.{version_minor}")
      return version_major, version_minor
  
  
  if __name__ == '__main__':
      f = open('test.las', 'rb')
      version = get_version(f)
      header = get_header(f, version)
      print(header.__dict__)
  
    
  ```
### 2. 读取点数据

- 新建`point.py`，定义不同点格式。由本文只需要提取xyz值，所有只需要xyz值，其他数据解析跳过
  
  ```python
  """
  定义点数据0~10格式解析
  """
  import struct
  
  
  class Point(object):
  
    def __init__(self, x_s_f, y_s_f, z_s_f, x_o, y_o, z_o):
        self.x_scale_factor = x_s_f
        self.y_scale_factor = y_s_f
        self.z_scale_factor = z_s_f
        self.x_offset = x_o
        self.y_offset = y_o
        self.z_offset = z_o

    def get_offset_bytes(self, point_data_record_format):
        """
        根据不同的点格式跳过的字节数
        :param point_data_record_format:点格式0~10
        :return:
        """
        # x,y,z共占12字节
        data_format = {
            0: 8,  # 点格式0 共20字节
            1: 16,  # 点格式1 共28字节
            2: 14,  # 点格式2 共26字节
            3: 22,  # 点格式3 共34字节
            4: 45,  # 点格式4 共57字节
            5: 51,  # 点格式5 共63字节
            6: 18,  # 点格式6 共30字节
            7: 24,  # 点格式7 共36字节
            8: 26,  # 点格式8 共38字节
            9: 47,  # 点格式9 共59字节
            10: 55,  # 点格式10 共67字节
        }

        offset_bytes = data_format.get(point_data_record_format, None)
        if offset_bytes is None:
            raise Exception(f"不存在当前的点格式{point_data_record_format}")
        return offset_bytes

    def read_point(self, f, offset_to_point_data, point_data_record_format, num):
        """
        读取当前文件中的点数据
        :param f:  数据文件
        :param offset_to_point_data: 点数据开始读取的地方
        :param point_data_record_format:  点数据格式0~10
        :param num: 读取多少个点
        :return: 读取的数据点
        """
        offset_bytes = self.get_offset_bytes(point_data_record_format)
        f.seek(offset_to_point_data)
        points = list()

        i = 0
        while i < num:
            point_bytes = f.read(12)
            x_record, y_record, z_record = struct.unpack_from('3l', point_bytes)
            x = x_record * self.x_scale_factor + self.x_offset
            y = y_record * self.y_scale_factor + self.y_offset
            z = z_record * self.z_scale_factor + self.z_offset
            i += 1
            f.read(offset_bytes)
            points.append((x, y, z))

        return points
  ```

- `test_read_las.py`中读取点数据
  
  ```python
  import struct
  from las.head import get_header
  from las.point import Point
  
  
  def get_version(f):
      f.read(24)
      version_major, version_minor = struct.unpack('2B', f.read(2))
      print(f"las版本:{version_major}.{version_minor}")
      return version_major, version_minor
  
  
  if __name__ == '__main__':
      f = open('test.las', 'rb')
      version = get_version(f)
      header = get_header(f, version)
      print(header.__dict__)
  
      points = Point(header.x_scale_factor,
                     header.y_scale_factor,
                     header.z_scale_factor,
                     header.x_offset,
                     header.y_offset,
                     header.z_offset,
                     )
  
      data = points.read_point(f, header.offset_to_point_data,
                               header.point_data_record_format,
                               header.number_of_point_records)
  
      print(data)
    
  ```


### 3. 输出XYZ坐标值到文件

在`test_read_las.py`将解析出来的点数据按照一定的格式（x,y,z）输出到txt文件。


```python
import struct
from las.head import get_header
from las.point import Point


def get_version(f):
    f.read(24)
    version_major, version_minor = struct.unpack('2B', f.read(2))
    print(f"las版本:{version_major}.{version_minor}")
    return version_major, version_minor


if __name__ == '__main__':
    f = open('test.las', 'rb')
    version = get_version(f)
    header = get_header(f, version)
    print(header.__dict__)

    points = Point(header.x_scale_factor,
                   header.y_scale_factor,
                   header.z_scale_factor,
                   header.x_offset,
                   header.y_offset,
                   header.z_offset,
                   )

    data = points.read_point(f, header.offset_to_point_data,
                             header.point_data_record_format,
                             header.number_of_point_records)

    f.close()

    with open("point.txt", 'a+') as f:
        for item in data:
            f.write(f'{item[0]},{item[1]},{item[2]}\n')

```

## 总结
- 本文完成了对las文件的读取并输出到文件
- 下一篇将讲解多进程分块加速读取las文件并输出到对应文件
