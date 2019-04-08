@[TOC](基于Flask与SQLAlchemy的单表接口)

tips:
 - 本文主要介绍基于Flask与SQLAlchemy的单表接口
 - 本文基于python3编写
 - 本文适合有一定Flask项目的朋友阅读
 - [代码仓库](https://github.com/qzq1111/flask-resful-example)

# 项目场景
一日，项目经理A找到我，先是表扬最近项目重构的不错，然后，提出一个单表接口想法。经过和他的仔细探讨，单表接口主要是实现一个数据表的增删改查，不会对其他的数据表造成干扰。

# 技术细节
- 实现http请求分发处理
- 实现请求参数识别和处理
- 实现数据库操作

# 实现逻辑
![实现逻辑](https://img-blog.csdnimg.cn/20190408163220551.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIyMDM0MzUz,size_16,color_FFFFFF,t_70)
# Flask MethodView
- Flask提供MethodView对每个 HTTP 方法执行不同的函数，即只需要修改不同函数处理方法。详见[官方文档](http://docs.jinkan.org/docs/flask/views.html#id4)
- 官方例子
1. 定义接口方法
	|URL	|HTTP 方法|描述
	|--|--|--|
	| /users/	|GET	|获得全部用户的列表
	| /users/	|POST	|创建一个新用户
	| /users/< id >	|GET	|显示某个用户
	| /users/< id >	|PUT	|更新某个用户
	| /users/< id >	|DELETE	|删除某个用户

2. 定义不同方法的处理逻辑
	```python
	from flask.views import MethodView
	
	class UserAPI(MethodView):
	
	    def get(self, user_id):
	        if user_id is None:
	            # return a list of users
	            pass
	        else:
	            # expose a single user
	            pass
	
	    def post(self):
	        # create a new user
	        pass
	
	    def delete(self, user_id):
	        # delete a single user
	        pass
	
	    def put(self, user_id):
	        # update a single user
	        pass
	```
3. 注册到路由
	```python
	user_view = UserAPI.as_view('user_api')
	app.add_url_rule('/users/', defaults={'user_id': None},
	                 view_func=user_view, methods=['GET',])
	app.add_url_rule('/users/', view_func=user_view, methods=['POST',])
	app.add_url_rule('/users/<int:user_id>', view_func=user_view,
	                 methods=['GET', 'PUT', 'DELETE'])
	```

# 编写字段解析类
1. 参数定义
    用法 **operator_modelFiled** ，即 下划线  前部分为**操作方法**，后部分为**model的键**。因此**所有的键都应与数据库表字段对应**。
    
	|  符号|含义  | 用法| 
	|--|--|--|
	| gt |大于 | gt_id=1|
	| ge |大于等于|ge_id=1|
	| lt |小于	 |lt_id=1|
	| le | 小于等于|le_id=1|
	| ne |不等于|ne_id=1|
	| eq |等于|eq_id=1|
	| ic |包含 |ic_id=1|
	| ni |不包含|ni_id=1|
	| in |查询多个相同字段的值| in_id=1\|2 |
	| by |排序(0:正序,1:倒序)|by_id=1|

2. 实现代码(base.py)
	```python 
	class BaseParse(object):
	    """
	    识别要查询的字段
	    """
	    __model__ = None
	    __request__ = request
	    by = frozenset(['by'])
	    query = frozenset(['gt', 'ge', 'lt', 'le', 'ne', 'eq', 'ic', 'ni', 'in'])
	
	    def __init__(self):
	        self._operator_funcs = {
	            'gt': self.__gt_model,
	            'ge': self.__ge_model,
	            'lt': self.__lt_model,
	            'le': self.__le_model,
	            'ne': self.__ne_model,
	            'eq': self.__eq_model,
	            'ic': self.__ic_model,
	            'ni': self.__ni_model,
	            'by': self.__by_model,
	            'in': self.__in_model,
	        }
	
	    def _parse_page_size(self):
	    	"""
	    	获取页码和获取每页数据量
	    	:return: page 页码
                 page_size 每页数据量
            """
	    	default_page = current_app.config['DEFAULT_PAGE_INDEX']
	    	default_size = current_app.config['DEFAULT_PAGE_SIZE']
	        page = self.__request__.args.get("page",default_page)
	        page_size = self.__request__.args.get("size",default_size )
	        page = int(page) - 1
	        page_size = int(page_size)
	        return page, page_size
	
	    def _parse_query_field(self):
	        """
	        解析查询字段
	        :return: query_field 查询字段
	                 by_field 排序字段
	        """
	        args = self.__request__.args
	        query_field = list()
	        by_field = list()
	        for query_key, query_value in args.items():
	            key_split = query_key.split('_', 1)
	            if len(key_split)!= 2:
	            	continue
	            operator, key = key_split
	            if not self._check_key(key=key):
	            	continue
	            if operator in self.query:
	            	data = self._operator_funcs[operator](key=key, value=query_value)
	            	query_field.append(data)
	            elif operator in self.by:
	            	data = self._operator_funcs[operator](key=key, value=query_value)
	                by_field.append(data)
	        return query_field, by_field
	
	    def _parse_create_field(self):
	        """
	        检查字段是否为model的字段,并过滤无关字段.
	        1.list(dict) => list(dict)
	        2. dict => list(dict)
	        :return:
	        """
	        obj = self.__request__.get_json(force=True)
	        if isinstance(obj, list):
	            create_field = list()
	            for item in obj:
	                if isinstance(item, dict):
	                    base_dict = self._parse_field(obj=item)
	                    create_field.append(base_dict)
	            return create_field
	        elif isinstance(obj, dict):
	            return [self._parse_field(obj=obj)]
	        else:
	            return list()
	
	    def _parse_field(self, obj=None):
	        """
	        检查字段模型中是否有，并删除主键值
	        :param obj:
	        :return:
	        """
	        obj = obj if obj is not None else self.__request__.get_json(force=True)
	        field = dict()
	        # 获取model主键字段
	        primary_key = map(lambda x: x.name, inspect(self.__model__).primary_key)
	
	        for key, value in obj.items():
	            if key in primary_key:
	                continue
	            if self._check_key(key):
	                field[key] = value
	        return field
	
	    def _check_key(self, key):
	        """
	        检查model是否存在key
	        :param key:
	        :return:
	        """
	        if hasattr(self.__model__, key):
	            return True
	        else:
	            return False
	
	    def __gt_model(self, key, value):
	        """
	        大于
	        :param key:
	        :param value:
	        :return:
	        """
	        return getattr(self.__model__, key) > value
	
	    def __ge_model(self, key, value):
	        """
	        大于等于
	        :param key:
	        :param value:
	        :return:
	        """
	        return getattr(self.__model__, key) > value
	
	    def __lt_model(self, key, value):
	        """
	        小于
	        :param key:
	        :param value:
	        :return:
	        """
	        return getattr(self.__model__, key) < value
	
	    def __le_model(self, key, value):
	        """
	        小于等于
	        :param key:
	        :param value:
	        :return:
	        """
	        return getattr(self.__model__, key) <= value
	
	    def __eq_model(self, key, value):
	        """
	        等于
	        :param key:
	        :param value:
	        :return:
	        """
	        return getattr(self.__model__, key) == value
	
	    def __ne_model(self, key, value):
	        """
	        不等于
	        :param key:
	        :param value:
	        :return:
	        """
	        return getattr(self.__model__, key) != value
	
	    def __ic_model(self, key, value):
	        """
	        包含
	        :param key:
	        :param value:
	        :return:
	        """
	        return getattr(self.__model__, key).like('%{}%'.format(value))
	
	    def __ni_model(self, key, value):
	        """
	        不包含
	        :param key:
	        :param value:
	        :return:
	        """
	        return getattr(self.__model__, key).notlike('%{}%'.format(value))
	
	    def __by_model(self, key, value):
	        """
	        :param key:
	        :param value: 0:正序,1:倒序
	        :return:
	        """
	        try:
	            value = int(value)
	        except ValueError as e:
	            logger.error(e)
	            return getattr(self.__model__, key).asc()
	        else:
	            if value == 1:
	                return getattr(self.__model__, key).asc()
	            elif value == 0:
	                return getattr(self.__model__, key).desc()
	            else:
	                return getattr(self.__model__, key).asc()
	
	    def __in_model(self, key, value):
	        """
	        查询多个相同字段的值
	        :param key:
	        :param value:
	        :return:
	        """
	        value = value.split('|')
	        return getattr(self.__model__, key).in_(value)
	
	```

# 编写数据库查询类
1. 定义
	根据项目实际需求，单表接口主要涉及到数据表的**增删改查**，主要有新建数据、批量新建数据、删除、更新、主键查询、多条件查询、分页、排序等数据库操作方法。

2. 实现代码(base.py)
	```python
	class BaseQuery(object):
	    """
	    查询方法
	    """
	    __model__ = None
	
	    def _find(self, query):
	    	"""
	    	根据查询参数获取内容
	    	"""
	        return self.__model__.query.filter(*query).all()
	
	    def _find_by_page(self, page, size, query, by):
	    	"""
	    	根据查询参数，分页，排序获取内容
	    	"""
	        base = self.__model__.query.filter(*query).order_by(*by)
	        cnt = base.count()
	        data = base.slice(page * size, (page + 1) * size).all()
	        return cnt, data
	
	    def _get(self, key):
	    	"""
	    	根据主键ID获取数据
	    	"""
	        return self.__model__.query.get(key)
	
	    def _create(self, args):
	    	"""
	    	创建新的一条数据或批量新建
	    	"""
	        for base in args:
	            model = self.__model__()
	            for k, v in base.items():
	                setattr(model, k, v)
	            db.session.add(model)
	        try:
	            db.session.commit()
	            return True
	        except Exception as e:
	            logger.error(e)
	            return False
	
	    def _update(self, key, kwargs):
	    	"""
	    	更新数据
	    	"""
	        model = self._get(key)
	        if model:
	            for k, v in kwargs.items():
	                setattr(model, k, v)
	            try:
	                db.session.add(model)
	                db.session.commit()
	                return True
	            except Exception as e:
	                logger.error(e)
	                return False
	        else:
	            return False
	
	    def _delete(self, key):
	    	"""
	    	删除数据
	    	"""
	        model = self._get(key)
	        if model:
	            try:
	                db.session.delete(model)
	                db.session.commit()
	                return True
	            except Exception as e:
	                logger.error(e)
	                return False
	        else:
	            return False
	
	    def parse_data(self, data):
	    	"""
	    	解析查询的数据
	    	"""
	        if data:
	            if isinstance(data, (list, tuple)):
	            
	                data = list(map(lambda x: {p.key: getattr(x, p.key)
	                                           for p in self.__model__.__mapper__.iterate_properties
	                                           }, data))
	            else:
	                data = {p.key: getattr(data, p.key) 
	                for p in self.__model__.__mapper__.iterate_properties}
	        return data
	```
# 编写Service类
1. 定义
	前面我们已经知道了Flask实现了不同http请求的分发(MethodView)，因此只需要自定义不同http请求处理逻辑。通过前面以及解决了参数解析或数据库操作。那么将BaseParse, BaseQuery, MethodView组合起来，就可以完成单表接口。
2. 实现代码(base.py)
	```python
	def view_route(f):
	    """
	    路由设置,统一返回格式
	    :param f:
	    :return:
	    """
	    def decorator(*args, **kwargs):
	        rv = f(*args, **kwargs)
	        if isinstance(rv, (int, float)):
	            res = ResMsg()
	            res.update(data=rv)
	            return jsonify(res.data)
	        elif isinstance(rv, tuple):
	            if len(rv) >= 3:
	                return jsonify(rv[0]), rv[1], rv[2]
	            else:
	                return jsonify(rv[0]), rv[1]
	        elif isinstance(rv, dict):
	            return jsonify(rv)
	        elif isinstance(rv, bytes):
	            rv = rv.decode('utf-8')
	            return jsonify(rv)
	        else:
	            return jsonify(rv)
	    return decorator
	    
	class Service(BaseParse, BaseQuery, MethodView):
	    __model__ = None
	    
	    # 装饰器控制数据返回格式
	    decorators = [view_route]
	
	    def get(self, key=None):
	        """
	        获取列表或单条数据
	        :param key:
	        :return:
	        """
	        res = ResMsg()
	        if key is not None:
	            data = self.parse_data(self._get(key=key))
	            if data:
	                res.update(data=data)
	            else:
	                res.update(code=ResponseCode.NO_RESOURCE_FOUND)
	        else:
	            query, by = self._parse_query_field()
	            page, size = self._parse_page_size()
	            cnt, data = self._find_by_page(page=page, size=size, query=query, by=by)
	            data = self.parse_data(data)
	            if data:
	                res.update(data=data)
	            else:
	                res.update(code=ResponseCode.NO_RESOURCE_FOUND)
	            res.add_field(name='total', value=cnt)
	            res.add_field(name='page', value=page + 1)
	            res.add_field(name='size', value=size)
	            res.update(data=data)
	        return res.data
	
	    def post(self):
	        """
	        创建数据
	        1.单条
	        2.多条
	        :return:
	        """
	        res = ResMsg()
	        data = self._parse_create_field()
	        if data:
	            if not self._create(args=data):
	                res.update(code=ResponseCode.FAIL)
	        else:
	            res.update(code=ResponseCode.INVALID_PARAMETER)
	        return res.data
	
	    def put(self, key=None):
	        """
	        更新某个数据
	        :return:
	        """
	        res = ResMsg()
	        if key is None:
	            res.update(code=ResponseCode.INVALID_PARAMETER)
	        else:
	            data = self._parse_field()
	            if not self._update(key=key, kwargs=data):
	                res.update(code=ResponseCode.FAIL)
	        return res.data
	
	    def delete(self, key=None):
	        """
	        删除某个数据
	        :return:
	        """
	        res = ResMsg()
	        if key is None:
	            res.update(code=ResponseCode.INVALID_PARAMETER)
	        elif not self._delete(key=key):
	            res.update(code=ResponseCode.FAIL)
	        return res.data
	```
#  单表接口使用
1. 数据库表(model.py)
	```python
	class Article(db.Model):
	    """
	    文章表
	    """
	    __tablename__ = 'article'
	    id = db.Column(db.Integer, autoincrement=True, primary_key=True)
	    title = db.Column(db.String(20), nullable=False)  # 文章标题
	    body = db.Column(db.String(255), nullable=False)  # 文章内容
	```
2. 接口服务(service.py)
	```python
	from base import Service
	from models import *
	
	class ArticleAPI(Service):
	    """
	    文章单表接口
	    """
	    __model__ = Article
	    __methods__ = ["GET", "POST", "DELETE", "PUT"]
	    service_name = 'article'
	```
3. 注册服务(app.py)
	```python
	from flask import Flask
	from service import ArticleAPI
	app = Flask(__name__)
	article_view = ArticleAPI.as_view('article_api')
	app.add_url_rule('/article/', defaults={'key': None},
	                 view_func=article_view , methods=['GET',])
	app.add_url_rule('/article/', view_func=article_view , methods=['POST',])
	app.add_url_rule('/article/<string:key>', view_func=article_view ,
	                 methods=['GET', 'PUT', 'DELETE'])
	if __name__ =='__main__':
		app.run()
	```
4. 测试
	- 单条新建 POST http://127.0.0.1:5000/article/
	发送数据：`{"title":"测试1","body":"测试1"}`
	返回：`{"code":0,"data":null,"lang":"zh_CN","msg":"成功"}`
	-  批量新建  POST http://127.0.0.1:5000/article/
	发送数据：`[{"title":"测试2","body":"测试2"},{"title":"测试3","body":"测试3"}]`
	返回：`{"code":0,"data":null,"lang":"zh_CN","msg":"成功"}`
	- 查询 GET http://127.0.0.1:5000/article/?eq_title=测试1
	返回：`{"code":0,"data":[{"body":"测试1","id":1,"title":"测试1"}],"lang":"zh_CN","msg":"成功", "page":1,"size":10,"total":1}`
	- 查询 GET http://127.0.0.1:5000/article/1
	返回：`{"code":0, "data":{"body":"测试1", "id":1,"title":"测试1" },"lang":"zh_CN","msg":"成功"}`
	- 更新 PUT http://127.0.0.1:5000/article/1
	发送数据：`{"title":"测试111","body":"测试111"}`
	返回：`{"code":0,"data":null,"lang":"zh_CN","msg":"成功"}`
	- 删除 DELETE  http://127.0.0.1:5000/article/1
	返回：`{"code":0,"data":null,"lang":"zh_CN","msg":"成功"}`
	  
# 总结
- 文中涉及到了字符串操作解析方面、类的定义、SQLAlchemy使用，Flask MethodView使用
- 通过类继承的方法，可以快速扩展单表接口，大大的提高了开发效率。如果有其他实现需求，也可以通过重写的方法完成定制。
- 下一篇文章将介绍reids在Flask里面的使用
