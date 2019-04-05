@[Toc](Flask 接口响应封装及自定义json返回类型)

***
tips:
- 本文主要解决统一响应文本封装及json响应文本类型错误问题
- 本文基于python3编写
- [代码仓库](https://github.com/mad7802004/flask-resful-example)

***
# 问题重现
- 前文[《Flask后端实践  连载三 接口标准化》](https://blog.csdn.net/qq_22034353/article/details/88701947)实现了响应文本的封装，即：
	```python
	from response import ResMsg
	@app.route("/", methods=["GET"])
	def test():
	    res = ResMsg()
	    test_dict = dict(name="zhang", age=18)
	    # 此处只需要填入响应状态码,即可获取到对应的响应消息
	    res.update(code=ResponseCode.SUCCESS, data=test_dict)
	    return jsonify(res.data)
	```
- 其中的`jsonify`是必不可少的，但是我们希望一个函数直接返回响应文本，实现`jsonify`的自动封装，不必重复书写，类似` return res.data`这样的格式。
- 当响应数据中存在`datetime`、`Decimal`等类型的时候，使用`jsonify`转换时会出错，报`TypeError: Object of type  {} is not JSON serializable`。
	python与json数据类型对应转换表：

	|python  |json  |
	|--|--|
	| dict | object |
	| list,tuple| array|
	| str| string|
	|int,float|float|
	|True|true|
	|False|false|
	|None|null|

***
# 统一封装，减少重复代码
1. Python装饰器
	Python装饰器在网上有许多教程，请读者自行学习装饰的原理以及用法。知乎上抄的这一段，装饰器本质上是一个Python函数，它可以让其他函数在不需要做任何代码变动的前提下增加额外功能，装饰器的返回值也是一个函数对象。它经常用于有切面需求的场景，比如：插入日志、性能测试、事务处理、缓存、权限校验等场景。装饰器是解决这类问题的绝佳设计，有了装饰器，我们就可以抽离出大量与函数功能本身无关的雷同代码并继续重用。概括的讲，装饰器的作用就是为已经存在的对象添加额外的功能。[知乎详解](https://www.zhihu.com/question/26930016)
	
2. 分析Flask 路由响应内容
[Flask视图函数](http://flask.pocoo.org/docs/1.0/quickstart/#about-responses)支持返回三个值：第一个是返回的数据，第二个是状态码，第三个是头部字典。默认第二个参数为200，第三个参数也
	```python
	@app.route('/')
	def test():
	    return 'Hello, World!', 200, {'X-Foo': 'bar'}
	```
   
3. 编写route装饰器（util.py）
	```python
	def route(bp, *args, **kwargs):
	    """
	    路由设置,统一返回格式
	    :param bp: 蓝图
	    :param args:
	    :param kwargs:
	    :return:
	    """
	    kwargs.setdefault('strict_slashes', False)
	
	    def decorator(f):
	        @bp.route(*args, **kwargs)
	        @wraps(f)
	        def wrapper(*args, **kwargs):
	            rv = f(*args, **kwargs)
	            # 响应函数返回整数和浮点型
	            if isinstance(rv, (int, float)):
	                res = ResMsg()
	                res.update(data=rv)
	                return jsonify(res.data)
	            # 响应函数返回元组
	            elif isinstance(rv, tuple):
	                # 判断是否为多个参数
	                if len(rv) >= 3:
	                    return jsonify(rv[0]), rv[1], rv[2]
	                else:
	                    return jsonify(rv[0]), rv[1]
	            # 响应函数返回字典
	            elif isinstance(rv, dict):
	                return jsonify(rv)
	            # 响应函数返回字节
	            elif isinstance(rv, bytes):
	                rv = rv.decode('utf-8')
	                return jsonify(rv)
	            else:
	                return jsonify(rv)
	
	        return f
	
	    return decorator

	```
4. 使用route装饰器(test.py)
	```python 
	from flask import Blueprint
	from util import route
	from response import ResMsg
	from code import ResponseCode
	
	bp = Blueprint(service_name, __name__, url_prefix="/")
	
	@route(bp, '/packed_response', methods=["GET"])
	def test_packed_response():
	    """
	    测试响应封装
	    :return:
	    """
	    res = ResMsg()
	    test_dict = dict(name="zhang", age=18)
	    # 此处只需要填入响应状态码,即可获取到对应的响应消息
	    res.update(code=ResponseCode.SUCCESS, data=test_dict)
	    # 此处不再需要用jsonify，如果需要定制返回头或者http响应如下所示
	    # return res.data,200,{"token":"111"}
	    return res.data
	```
***
# 解决json类型错误
1. Flask json转换类如下，只需要我们重新写default函数，定义转换规则，便能到达我们想要的效果。
	```python
	class JSONEncoder(_json.JSONEncoder):
	    """The default Flask JSON encoder.  This one extends the default simplejson
	    encoder by also supporting ``datetime`` objects, ``UUID`` as well as
	    ``Markup`` objects which are serialized as RFC 822 datetime strings (same
	    as the HTTP date format).  In order to support more data types override the
	    :meth:`default` method.
	    """

	    def default(self, o):
	        """Implement this method in a subclass such that it returns a
	        serializable object for ``o``, or calls the base implementation (to
	        raise a :exc:`TypeError`).
	
	        For example, to support arbitrary iterators, you could implement
	        default like this::
	
	            def default(self, o):
	                try:
	                    iterable = iter(o)
	                except TypeError:
	                    pass
	                else:
	                    return list(iterable)
	                return JSONEncoder.default(self, o)
	        """
	        if isinstance(o, datetime):
	            return http_date(o.utctimetuple())
	        if isinstance(o, date):
	            return http_date(o.timetuple())
	        if isinstance(o, uuid.UUID):
	            return str(o)
	        if hasattr(o, '__html__'):
	            return text_type(o.__html__())
	        return _json.JSONEncoder.default(self, o)
	```
2. 自定义Flask json解析类(core.py)
	```python
	import datetime
	import decimal
	import uuid
	
	from flask.json import JSONEncoder as BaseJSONEncoder
	
	
	class JSONEncoder(BaseJSONEncoder):
		"""
	    重新default方法，支持更多的转换方法
	    """
	    def default(self, o):
	        """
	        如有其他的需求可直接在下面添加
	        :param o: 
	        :return:
	        """
	        if isinstance(o, datetime.datetime):
	            # 格式化时间
	            return o.strftime("%Y-%m-%d %H:%M:%S")
	        if isinstance(o, datetime.date):
	            # 格式化日期
	            return o.strftime('%Y-%m-%d')
	        if isinstance(o, decimal.Decimal):
	            # 格式化高精度数字
	            return str(o)
	        if isinstance(o, uuid.UUID):
	            # 格式化uuid
	            return str(o)
	        if isinstance(o, bytes):
	            # 格式化字节数据
	            return o.decode("utf-8")
	        return super(JSONEncoder, self).default(o)
	```
3. 使用自定义Flask json解析类(app.py)
	```python
	from flask import Flask
	from core import JSONEncoder
	
	app = Flask(__name__)
	
	# 返回json格式转换
    app.json_encoder = JSONEncoder
	
	if __name__ == "__main__":
	    app.run()
	```
4. 测试(test.py)
	```python
	from datetime import datetime
	from decimal import Decimal
	from flask import Blueprint
	from util import route
	from response import ResMsg
	from code import ResponseCode
	
	bp = Blueprint(service_name, __name__, url_prefix="/")
	
	@route(bp, '/type_response', methods=["GET"])
	def test_type_response():
	    """
	    测试返回不同的类型
	    :return:
	    """
	    res = ResMsg()
	    now = datetime.now()
	    date = datetime.now().date()
	    num = Decimal(11.11)
	    test_dict = dict(now=now, date=date, num=num)
	    # 此处只需要填入响应状态码,即可获取到对应的响应消息
	    res.update(code=ResponseCode.SUCCESS, data=test_dict)
	    # 此处不再需要用jsonify，如果需要定制返回头或者http响应如下所示
	    # return res.data,200,{"token":"111"}
	    return res.data

	```
	响应消息：
	```
		{
	    "code":0,
	    "data":{
	        "date":"2019-03-27",
	        "now":"2019-03-27 11:05:40",
	        "num":"11.1099999999999994315658113919198513031005859375"
	    },
	    "lang":"zh_CN",
	    "msg":"成功"
	}
	```
***
# 总结
- 利用装饰器和重写类的方法，解决了前文提出的问题。
- 下一篇文章将介绍Flask与SQLAlchemy的集成和简单使用。
