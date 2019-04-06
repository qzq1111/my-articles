@[Toc](Flask yaml配置logger及logger使用)

 
tips: 
- 上一篇文章[《Flask后端实践  连载一 加载yaml配置文件》](https://blog.csdn.net/qq_22034353/article/details/88591681)主要介绍了Flask如何加载yaml文件配置，这篇文章则是进一步使用yaml文件来配置logger及logger的使用。
- 本文基于python3编写
- [代码仓库](https://github.com/mad7802004/flask-resful-example)
 
# Python logging模块
- Logging日志一共五个级别

 1. **debug** : 打印全部的日志,详细的信息,通常只出现在诊断问题上
 2. **info** : 打印info,warning,error,critical级别的日志,确认一切按预期运行
 3. **warning** : 打印warning,error,critical级别的日志,一个迹象表明,一些意想不到的事情发生了,或表明一些问题在不久的将来(例如。磁盘空间低”),这个软件还能按预期工作
 4. **error** : 打印error,critical级别的日志,更严重的问题,软件没能执行一些功能
 5. **critical** : 打印critical级别,一个严重的错误,这表明程序本身可能无法继续运行
 6. 日志级别：**CRITICAL** >**ERROR**> **WARNING** > **INFO**> **DEBUG**> **NOTSET**

- logging日志设置以及使用
```python
import logging

# 日志记录最低输出级别设置
logger = logging.getLogger()    
logger.setLevel(logging.DEBUG) 

# Logging记录到文件
fh = logging.FileHandler("test.log", mode='w')
logger.addHandler(fh)  # 将logger添加到handler里面

# Logging记录格式以及设置
formatter = logging.Formatter("%(asctime)s - %(message)s") # 记录时间和消息
fh.setFormatter(formatter) # 此处设置handler的输出格式

# Logging使用
logger.info("this is info")
logger.debug("this is debug")
logger.warning("this is warning")
logging.error("this is error")
logger.critical("this is critical")
```
- 输出test.log
```python
2019-03-18 21:39:37,260 - this is info
2019-03-18 21:39:37,260 - this is debug
2019-03-18 21:39:37,260 - this is warning
2019-03-18 21:39:37,260 - this is error
2019-03-18 21:39:37,260 - this is critical
```
- 查看更多[详细信息](http://python.jobbole.com/86887/)
 
# Flask自带logger
Flask在0.3版本后就有了日志工具logger。
- Flask[日志配置](https://dormousehole.readthedocs.io/en/latest/logging.html#logging)
- 日志工具简单使用：
```python
from flask import Flask

app = Flask(__name__)
app.logger.debug('A value for debugging')
app.logger.warning('A warning occurred (%d apples)', 42)
app.logger.error('An error occurred')

if __name__ == "__main__":
    app.run()
```
- 两种不同的方式配置logger
	- 第一种，直接设置 
	```python
	import logging
	from flask import Flask
	app = Flask(__name__)
    handler = logging.FileHandler(filename="test.log", encoding='utf-8')
    handler.setLevel("DEBUG")
    format_ ="%(asctime)s[%(name)s][%(levelname)s] :%(levelno)s: %(message)s"
    formatter = logging.Formatter(format_)
    handler.setFormatter(formatter)
    app.logger.addHandler(handler)
    if __name__ == "__main__":
    	app.run()
	```
	-  第二种，载入配置字典
	```python
	import logging
	import logging.config
	from flask import Flask
	logger_conf ={
	    'version': 1,
	    'formatters': {'default': {
	        'format': '[%(asctime)s] %(levelname)s in %(module)s: %(message)s',
	    }},
	    'handlers': {'wsgi': {
	        'class': 'logging.StreamHandler',
	        'stream': 'ext://flask.logging.wsgi_errors_stream',
	        'formatter': 'default'
	    }},
	    'root': {
	        'level': 'INFO',
	        'handlers': ['wsgi']
	    }
	 }
	
	app = Flask(__name__)
    logging.config.dictConfig(logger_conf)
    if __name__ == "__main__":
    	app.run()
	```
- 工厂模式下不同地方使用logger，请使用**current_app.logger**
```python
import logging
import logging.config
from flask import current_app, Flask

logger_conf ={
    'version': 1,
    'formatters': {'default': {
        'format': '[%(asctime)s] %(levelname)s in %(module)s: %(message)s',
    }},
    'handlers': {'wsgi': {
        'class': 'logging.StreamHandler',
        'stream': 'ext://flask.logging.wsgi_errors_stream',
        'formatter': 'default'
    }},
    'root': {
        'level': 'INFO',
        'handlers': ['wsgi']
    }
}

def create_app():
    app = Flask(__name__)
    # 方法一日志设置
    handler = logging.FileHandler(filename="test.log", encoding='utf-8')
    handler.setLevel("DEBUG")
    format_ ="%(asctime)s[%(name)s][%(levelname)s] :%(levelno)s: %(message)s"
    formatter = logging.Formatter(format_)
    handler.setFormatter(formatter)
    app.logger.addHandler(handler)
    # 方法二日志设置
    # logging.config.dictConfig(logger_conf)
    return app
    
app = create_app()

@app.route("/", methods=["GET"])
def test():
    current_app.logger.info("this is info")
    current_app.logger.debug("this is debug")
    current_app.logger.warning("this is warning")
    current_app.logger.error("this is error")
    current_app.logger.critical("this is critical")
    return "ok"


if __name__ == "__main__":
    app.run()
```
请求http://127.0.0.1:5000  输出到test.log内容
```python
2019-03-18 22:02:58,718[flask.app][WARNING] :30: this is warning
2019-03-18 22:02:58,718[flask.app][ERROR] :40: this is error
2019-03-18 22:02:58,719[flask.app][CRITICAL] :50: this is critical
2019-03-18 22:04:03,991[flask.app][WARNING] :30: this is warning
2019-03-18 22:04:03,992[flask.app][ERROR] :40: this is error
2019-03-18 22:04:03,992[flask.app][CRITICAL] :50: this is critical
```
- 对于以上的Flask logger的基本配置都能满足基本的日常需求。但是如果同时配置多个不同的handler，并且将不同内容的日志输出到不同的文件会很麻烦。所以需要一个统一的日志配置文件来处理。
 
# yaml配置logger
1. 在前文中，第二种配置logger的方法中使用了字典，在上一篇文章[《Flask如何加载yaml文件配置》](https://blog.csdn.net/qq_22034353/article/details/88591681)中介绍了python读取yaml文件以及yaml文件在python中的转换。所以只要按照一定的规则编写yaml文件，加载到日志配置中，即可完成配置。
2. 文件目录
	```
	-----app
	│    │    
	│    factoty.py
	-----config
	|    |
	|    config.yaml
	|    logging.yaml
	run.py
	```
4. 编写logging.yaml
	```yaml
	version: 1
	disable_existing_loggers: False
	# 定义日志输出格式，可以有多种格式输出
	formatters:
		simple:
	    	format: "%(message)s"
	    error:
	        format: "%(asctime)s [%(name)s] [%(levelname)s] :%(levelno)s: %(message)s"
	
	# 定义不同的handler，输出不同等级的日志消息
	handlers:
	    console:
	        class: logging.StreamHandler # 输出到控制台
	        level: DEBUG
	        formatter: simple
	        stream: ext://flask.logging.wsgi_errors_stream # 监听flask日志
	    info_file_handler:
	        class: logging.handlers.RotatingFileHandler # 输出到文件
	        level: INFO
	        formatter: simple
	        filename: ./logs/info.log
	        maxBytes: 10485760 # 10MB
	        backupCount: 20 #most 20 extensions
	        encoding: utf8
	    error_file_handler:
	        class: logging.handlers.RotatingFileHandler # 输出到文件
	        level: ERROR
	        formatter: error
	        filename: ./logs/errors.log
	        maxBytes: 10485760 # 10MB
	        backupCount: 20
	        encoding: utf8
	# 启用handler
	root:
	    level: INFO
	    handlers: [console,info_file_handler,error_file_handler]
	```
5. 编写Flask config.yaml
	```yaml
	COMMON: &common
	  # app设置
	  DEBUG: False
	  TESTING: False
	  THREADED: False
	  SECRET_KEY: insecure
	  # 日志配置文件路径
	  LOGGING_CONFIG_PATH: ./config/logging.yaml
	  # 日志文件存放位置
	  LOGGING_PATH: ./logs
	  
	DEVELOPMENT: &development
	  <<: *common
	  DEBUG: True
	  ENV:  dev
	TESTING: &testing
	  <<: *common
	  ENV: test
	  TESTING: True
	  
	PRODUCTION: &production
	  <<: *common
	  ENV: prod
	  SECRET_KEY: shouldbereallysecureatsomepoint
	
	```
6. 加载Flask配置及日志配置
- 工厂文件（factoty.py）
	```python
	import logging
	import logging.config
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
	    if not os.path.exists(app.config['LOGGING_PATH']):
	        # 日志文件目录
	        os.mkdir(app.config['LOGGING_PATH'])
	    # 日志设置
	    with open(app.config['LOGGING_CONFIG_PATH'], 'r', encoding='utf-8') as f:
	        dict_conf = yaml.safe_load(f.read())
	    logging.config.dictConfig(dict_conf) # 载入日志配置
	    return app
	    
	def read_yaml(config_name, config_path):
	    """
	    config_name:需要读取的配置内容
	    config_path:配置文件路径
	    """
	    if config_name and config_path:
	        with open(config_path, 'r', encoding='utf-8') as f:
	            conf = yaml.safe_load(f.read())
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
 

# logger在蓝图(Blueprint)中的使用

1. 第一种方法(test.py)，使用flask内置的handler输出日志，返回名为`flask.app`的日志。
	```python 
	from flask import current_app,Blueprint
	bp = Blueprint("test", __name__, url_prefix='/test')
	
	@bp.route('',methods=["GET"])
	def test():
		current_app.logger.info("this is info")
	    current_app.logger.debug("this is debug")
	    current_app.logger.warning("this is warning")
	    current_app.logger.error("this is error")
	    current_app.logger.critical("this is critical")
	    return "ok"
	
	# 输出日志
	# 2019-03-18 22:02:58,718[flask.app][WARNING] :30: this is warning
	# 2019-03-18 22:02:58,718[flask.app][ERROR] :40: this is error
	# 2019-03-18 22:02:58,719[flask.app][CRITICAL] :50: this is critical
	# 2019-03-18 22:04:03,991[flask.app][WARNING] :30: this is warning
	# 2019-03-18 22:04:03,992[flask.app][ERROR] :40: this is error
	# 2019-03-18 22:04:03,992[flask.app][CRITICAL] :50: this is critical
	```
2. 第二种方法(test.py)，输出日志是在某个文件产生的。
	```python
	import logging
	from flask import Blueprint
	
	bp = Blueprint("test", __name__, url_prefix='/test')
	logger = logging.getLogger(__name__) #返回一个新的以文件名为名的logger
	
	@bp.route('',methods=["GET"])
	def test():
		logger.info("this is info")
	    logger.debug("this is debug")
	    logger.warning("this is warning")
	    logger.error("this is error")
	    logger.critical("this is critical")
	    return "ok"
	    
	# 输出日志
	# 2019-03-20 22:05:44,705 [app.test] [ERROR] :40: this is error
	# 2019-03-20 22:05:44,705 [app.test] [CRITICAL] :50: this is critical
	```
 

# 总结
- 本文主要介绍了Python logging模块的简单使用，读者可根据自身需要进行深入学习。
- 本文介绍Flask的官方日志配置配置方法以及工厂模式下蓝图如何使用Flask自带logger。
- 实现yaml文件配置Flask日志配置，以及两种不同的输出日志选项（Flask自带logger及文件名logger）。
-  下一篇将介绍如何设计实现FlaskRestful标准化接口。
 
