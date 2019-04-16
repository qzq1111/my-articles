# Flask使用redis数据库

tips:
 - 本文简单介绍Flask中使用redis
 - 本文代码基于python3编写
 - [代码仓库](https://github.com/qzq1111/flask-resful-example)

## 项目场景
在实际项目中，不频繁变化且重复使用的数据、有一定时效的数据等。放入redis中，不仅可以提高查询效率，还能减少维护成本。实际应用比如手机验证码，token验证、任务调度等。
## redis
1. 定义
REmote DIctionary Server(Redis) 是一个由Salvatore Sanfilippo写的key-value存储系统。Redis是一个开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。它通常被称为数据结构服务器，因为值（value）可以是 字符串(String), 哈希(Hash), 列表(list), 集合(sets) 和 有序集合(sorted sets)等类型。
2. 安装
此处[下载](https://github.com/MicrosoftArchive/redis/releases)对应系统的安装包，运行安装包即可。具体过程可以在[菜鸟教程](http://www.runoob.com/redis/redis-install.html)查看。
3. 使用方法
 [菜鸟教程](http://www.runoob.com/redis/redis-commands.html)查看
## Python使用redis
1. 安装Python redis包`pip install redis`
2. 简单使用
	```python
	
	import redis

	# 获取redis数据库连接
	r = redis.StrictRedis(host="127.0.0.1", port=6379, db=0)
	
	# redis存入键值对
	r.set(name="key", value="value")
	# 读取键值对
	print(r.get("key"))
	# 删除
	print(r.delete("key"))
	
	# redis存入Hash值
	r.hset(name="name", key="key1", value="value1")
	r.hset(name="name", key="key2", value="value2")
	# 获取所有哈希表中的字段
	print(r.hgetall("name"))
	# 获取所有给定字段的值
	print(r.hmget("name", "key1", "key2"))
	# 获取存储在哈希表中指定字段的值。
	print(r.hmget("name", "key1"))
	# 删除一个或多个哈希表字段
	print(r.hdel("name", "key1"))
	
	# 过期时间
	r.expire("name", 60)  # 60秒后过期
	
	# 更多相关内容可以参考菜鸟教程

	```
## Flask使用redis
1. 封装redis方法(util.py)
	```python
	from flask import current_app
	import redis 
	
	class Redis(object):
	    """
	    redis数据库操作
	    """

	    @staticmethod
	    def _get_r():
	    	host = current_app.config['REDIS_HOST']
	    	port=current_app.config['REDIS_PORT']
	    	db=current_app.config['REDIS_DB']
	        r = redis.StrictRedis(host, port,db)
	        return r
	
	    @classmethod
	    def write(cls, key, value, expire=None):
	    	"""
	    	写入键值对
	    	"""
	    	# 判断是否有过期时间，没有就设置默认值
	    	if expire:
	    		expire_in_seconds = expire
	    	else:
	    		expire_in_seconds = current_app.config['REDIS_EXPIRE']
	        r = cls._get_r()
	        r.set(key, value, ex=expire_in_seconds)
	
	    @classmethod
	    def read(cls, key):
	    	"""
	    	读取键值对内容
	    	"""
	        r = cls._get_r()
	        value = r.get(key)
	        return value.decode('utf-8') if value else value
	
	    @classmethod
	    def hset(cls, name, key, value):
	    	"""
	    	写入hash表
	    	"""
	        r = cls._get_r()
	        r.hset(name, key, value)
	
	    @classmethod
	    def hmset(cls, key, *value):
	    	"""
	    	读取指定hash表的所有给定字段的值
	    	"""
	        r = cls._get_r()
	        value = r.hmset(key, *value)
	        return value
	
	    @classmethod
	    def hget(cls, name, key):
	    	"""
	    	读取指定hash表的键值
	    	"""
	        r = cls._get_r()
	        value = r.hget(name, key)
	        return value.decode('utf-8') if value else value
	
	    @classmethod
	    def hgetall(cls, name):
	    	"""
	    	获取指定hash表所有的值
	    	"""
	        r = cls._get_r()
	        return r.hgetall(name)
	
	    @classmethod
	    def delete(cls, *names):
	        """
	        删除一个或者多个
	        """
	        r = cls._get_r()
	        r.delete(*names)
	
	    @classmethod
	    def hdel(cls, name, key):
	        """
			删除指定hash表的键值
	        """
	        r = cls._get_r()
	        r.hdel(name, key)
	
	    @classmethod
	    def expire(cls, name, expire=None):
	        """
	        设置过期时间
	        """
	        if expire:
	    		expire_in_seconds = expire
	    	else:
	    		expire_in_seconds = current_app.config['REDIS_EXPIRE']
	        r = cls._get_r()
	        r.expire(name, expire_in_seconds)
	```
2. 测试使用(test.py)
	```python
	from util import Redis
	bp = Blueprint(service_name, __name__, url_prefix="/")
	
	@bp.route('/testRedisWrite', methods=['GET'])
	def test_redis_write():
		"""
		测试redis
		"""
		Redis.write("test_key","test_value",60)
		return "ok"
	
	@bp.route('/testRedisRead', methods=['GET'])
	def test_redis_read():
		"""
		测试redis
		"""
		data = Redis.read("test_key")
		return data
	```
3. Flask App(app.py)
	```python
	from flask import Flask
	from test import bp
	app = Flask(__name__)
	app.config['REDIS_HOST'] = "127.0.0.1" # redis数据库地址
	app.config['REDIS_PORT'] = 6379 # redis 端口号
	app.config['REDIS_DB'] = 0 # 数据库名
	app.config['REDIS_EXPIRE'] = 60 # redis 过期时间60秒
	# 注册接口
    app.register_blueprint(bp)
    
	if __name__=="__main__":
		app.run()
	```
4. 启动Flask app
	4.1 访问 `http://127.0.0.1:5000/testRedisWrite` 返回 `"ok"`
	4.2 访问 `http://127.0.0.1:5000/testRedisRead` 返回 `test_value`
	
## 总结
- 简单的使用了redis以及相关的API封装，方便快捷的使用。
- 接下来的一篇文章，将介绍docker+gunicorn+nginx部署Flask后端的相关知识

