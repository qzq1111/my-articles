@[TOC](Flask 加载yaml配置文件)

***
tips:
- 本文代码基于python3
- [代码仓库](https://github.com/mad7802004/flask-resful-example)
***
# 项目场景
某天，项目经理A要求所有的配置文件需要可以配置，并且需要用yaml格式的文件进行配置。方便以后对配置文件的修改以及读取不同的配置内容，而不是在代码中修改配置信息。
***

# 什么是yaml文件？
  YAML是一种直观的能够被电脑识别的的数据序列化格式，容易被人类阅读，并且容易和脚本语言交互。YAML类似于XML，JSON等，但是语法简单得多，对于转化成数组或可以hash的数据时是很简单有效的。[详细了解](https://en.wikipedia.org/wiki/YAML)
***
# 如何编写yaml文件？
```yaml
# yaml格式
name: 张三
age: 37
children:
 - name: 小明
   age: 15
 - name: 小红
   age: 12
```
上面这段代码在python中具体输出如下：
```python
{
'name': '张三', 'age': 37, 
'children': [{'name': '小明', 'age': 15}, {'name': '小红', 'age': 12}]
}
```
从上面可以看出，通过yaml文件的简单书写，便能转换为python中的字典以及列表字典。
yaml文件编写规则:
- [英文](https://yaml.org/)
- [中文](https://blog.csdn.net/vincent_hbl/article/details/75411243)
***
# Flask加载yaml配置文件
- 创建虚拟环境，启用虚拟环境
- 安装PyYAML `pip install PyYAML`
- 安装Flask `pip install Flask`

编写一个配置文件（setting.yaml）如下：
```yaml
COMMON: &common
  SECRET_KEY: insecure
  DEBUG: False
  
DEVELOPMENT: &development
  <<: *common
  DEBUG: True
  
STAGING: &staging
  <<: *common
  SECRET_KEY: sortasecure

PRODUCTION: &production
  <<: *common
  SECRET_KEY: mdd1##$$%^!DSA#FDSF
```

编写读取yaml文件函数（read_yaml）
```python
import yaml
def read_yaml(yaml_file_path):
    with open(yaml_file_path, 'rb') as f:
        cf= f.read()
    cf = yaml.load(cf)
return cf
```
flask加载yaml文件配置

```python
from flask import Flask
app = Flask(__name__)
cf = read_yaml("setting.yaml")
app.config.update(cf)
```
完整代码(app.py)
```python
from flask import Flask
import yaml
def read_yaml(yaml_file_path):
    with open(yaml_file_path, 'rb') as f:
        cf= yaml.safe_load(f.read()) # yaml.load(f.read())
return cf

app = Flask(__name__)
cf = read_yaml("setting.yaml")
app.config.update(cf)

if __name__ == "__main__":
    app.run()
```
***
# 工厂模式

- 文件目录
```
-----app
│    │    
│    factoty.py
-----config
|    |
|    config.yaml
run.py
```
- 配置文件（config.yaml）
```yaml
COMMON: &common #标识
  DEBUG: False
  SECRET_KEY: insecure

DEVELOPMENT: &development
  <<: *common # 继承common，没有重新定义的变量，使用common变量值
  DEBUG: True
  
STAGING: &staging
  <<: *common
  SECRET_KEY: sortasecure

PRODUCTION: &production
  <<: *common
  SECRET_KEY: mdd1##$$%^!DSA#FDSF
```
- 工厂文件（factory.py）
```python
import yaml
import os
from flask import Flask
def create_app(config_name=None, config_path=None):
	app = Flask(__name__)
    # 读取配置文件
    if not config_path:
        pwd = os.getcwd()
        config_path = os.path.join(pwd, 'config/config.yaml')
    if not config_name:
        config_name = 'PRODUCTION'
    conf = read_yaml(config_name, config_path)
    app.config.update(conf)
    return app
    
def read_yaml(config_name,config_path):
	""" 
	config_name:需要读取的配置内容
	config_path:配置文件路径
	"""
	if config_name and config_path:
        with open(config_path, 'r') as f:
            conf =  yaml.safe_load(f.read()) # yaml.load(f.read())
        if config_name in conf.keys():
            return conf[config_name.upper()]
        else:
            raise KeyError('未找到对应的配置信息')
    else:
        raise ValueError('请输入正确的配置名称或配置文件路径')
```
- 运行文件(run.py)
```python
from app import factory
app = factory.create_app(config_name='DEVELOPMENT')

if __name__ == "__main__":
    app.run()
```
***
# 总结
-  yaml编写配置文件，简单高效。
-  编写不同的配置内容，方便生产开发测试使用。
-  目前实现了flask加载yaml配置文件，后续的文章还有更多关于yaml文件配置flask内容。
