# Flask与SQLAlchemy的集成和简单使用


tips:
 - 本文主要介绍Flask与SqlAlchemy的集成和简单使用
 - 本文基于python3编写
 - [代码仓库](https://github.com/mad7802004/flask-resful-example)
 
  
 ## 前言
 - 在实际的生产中，往往都需要数据库来做数据存储以及信息交互。
 - 本文选用SQLAlchemy作为orm工具，可以减少sql代码编写以及防止sql注入的风险的存在。
 - 本文使用免费的mysql数据库。
  
 
 ## Flask与SQLAlchemy结合
 1. 安装flask-sqlalchemy  `pip install  flask-sqlalchemy`
 2. 安装pymysql  `pip install pymysql`
 3. 实例化SQLAlchemy(core.py)
	```python 
	from flask_sqlalchemy import SQLAlchemy
	
	db = SQLAlchemy()
	```
 4. 注册到flask App中(app.py)并配置数据库连接
	```python
	from flask import Flask
	from core import db
	app=Flask(__name__)
	
    USERNAME = 'root' # 用户名
    PASSWORD = 'mad123'# 密码
    HOST = '127.0.0.1' # 数据库地址
    PORT = '3306' # 端口
    DATABASE = 'test' # 数据库名
    database_url ='mysql+pymysql://{}:{}@{}:{}/{}?charset=utf8'.format(
        USERNAME,PASSWORD,HOST,PORT,DATABASE)
    #添加数据库配置文件到flask App中
	app.config['SQLALCHEMY_DATABASE_URI'] = database_url 
	app.config['SQLALCHEMY_COMMIT_ON_TEARDOWN'] = False
	# 注册数据库连接
	db.app =app
	db.init_app(app)

	```
 
 
## 定义model及创建数据库表
- 定义model(model.py)
	```python
	from core import db
	
	# 新建用户
	class User(db.Model):
	    """
	    用户表
	    """
	    __tablename__ = 'user'
	    id = db.Column(db.Integer, autoincrement=True, primary_key=True)
	    name = db.Column(db.String(20), nullable=False)  # 用户姓名
	    age = db.Column(db.Integer, nullable=False)  # 用户年龄
	
	# 新建文章model
	class Article(db.Model):
	    """
	    文章表
	    """
	    __tablename__ = 'article'
	    id = db.Column(db.Integer, autoincrement=True, primary_key=True)
	    title = db.Column(db.String(20), nullable=False)  # 文章标题
	    body = db.Column(db.String(255), nullable=False)  # 文章内容
	    last_change_time = db.Column(db.DateTime, nullable=False) # 最后一次修改日期
	    author_id = db.Column(db.Integer, db.ForeignKey('user.id'))  # 作者
	    author = db.relationship('User', backref=db.backref('articles'))
	
	# 新建文章修改日志 
	class ChangeLogs(db.Model):
	    """
	    修改日志
	    """
	    __tablename__ = 'change_logs'
	    id = db.Column(db.Integer, autoincrement=True, primary_key=True)
	    author_id = db.Column(db.Integer, db.ForeignKey('user.id'))  # 作者
	    article_id = db.Column(db.Integer, db.ForeignKey('article.id'))  # 文章
	    modify_content = db.Column(db.String(255), nullable=False)  # 修改内容
	    create_time = db.Column(db.DateTime, nullable=False) # 创建日期

	```
- 创建数据库表  执行`db.create_all()`命令，需要在启动FlaskApp之后再命令行执行

 
## 增
1. 单个新建
	```python
	# 方法一,声明实例后赋值
	new_user = User()
	new_user.name = 'qin'
	new_user.age =23
	db.session.add(new_user)
	db.session.commit()
	# 方法二,声明实例直接赋值
	new_user = User(name='zhang',age=26)
	db.session.add(new_user)
	db.session.commit()
	# 方法三，利用字典赋值
	user_dict=dict(name='xiong',age=24)
	new_user = User(**user_dict)
	db.session.add(new_user)
	db.session.commit()
	```
	
2. 批量新建
	```python
	# 方法一:声明多个实例，逐条插入
	new_user_qin = User(name='qin',age=26)
	new_user_zhang = User(name='zhang',age=26)
	new_user_xiong = User(name='xiong',age=24)
	
	db.session.add_all([new_user_qin ,new_user_zhang,new_user_xiong ])
	db.session.commit()
	# 方法二：一次性插入
	new_users = [
	dict(name='qin',age=24),
	dict(name='zhang',age=24),
	dict(name='xiong',age=24)]
	
	db.session.execute(User.__table__.insert(), new_users )
    db.session.commit()
	```
 
## 查
1. 查询一个元素
	```python
	# 方法一,first()  
	user = db.session.query(User).filter(User.id == 1).first() 
	user = User.query.filter(User.id ==1).first()
	
	# 方法二,get() 此方法用于主键查询
	user = db.session.query(User).get(1)  
	user = User.query.get(1)
	```
2. 查询所有元素
	```python
	users = db.session.query(User).all()
	users = User.query.all()
	```
3. 查询条件过滤 filter
	```python
	# 等于
	users = db.session.query(User).filter(User.id ==1).all()
	users = User.query.filter(User.id==1).all()
	# 大于等于
	users = db.session.query(User).filter(User.id >=1).all()
	users = User.query.filter(User.id>=1).all()
	# 小于等于
	users = db.session.query(User).filter(User.id <=1).all()
	users = User.query.filter(User.id<=1).all()
	# 不等于
	users = db.session.query(User).filter(User.id !=1).all()
	users = User.query.filter(User.id !=1).all()
	# 两者之间
	users = db.session.query(User).filter(User.id.between(1,10)).all()
	users = User.query.filter(User.id.between(1,10)).all()
	# 模糊查询
	users = db.session.query(User).filter(User.name.like('%qin%')).all()
	users = User.query.filter(User.name.like('%qin%')).all()
	# 多条件查询
	users = db.session.query(User).filter(User.id ==1, User.age==10).all()
	users = User.query.filter(User.id>=1, User.age==10).all()
	```
4. 多表关联查询 
	```python
	# 外键关联,由于只存在一个ForeignKey, SQLAlchemy知道如何选取合适的列进行JOIN。
	# 如果没有定义ForeignKey,或者存在多个，此时你需要手动指明你参与JOIN的列
	article_user = User.query.join(Article).filter(Article.id == 1).first()
	article_user = db.session.query(User).\
	join(Article).filter(Article.id == 1).first()
	
	# 指定键关联，查询某篇文章作者
	article_user = User.query.join(Article,Article.author_id == User.id).\
	filter(Article.id == 1).first()
	article_user = db.session.query(User).\
	join(Article,Article.author_id == User.id).filter(Article.id == 1).first()
	
	# 多键关联，查询某篇文章的修改内容
	change_logs = ChangeLogs.query.join(User, ChangeLogs.author_id == User.id). \
	    join(Article, Article.id == ChangeLogs.article_id). \
	    filter(Article.id == 1, User.id == 1).all()
	change_logs = db.session.query(ChangeLogs). \
	    join(User, ChangeLogs.author_id == User.id). \
	    join(Article, Article.id == ChangeLogs.article_id). \
	    filter(Article.id == 1, User.id == 1).all()
	
	# 筛选多个表的字段，查询某篇文章的所有修改内容
	change_logs = db.session.query(ChangeLogs.modify_content,
                               ChangeLogs.create_time,
                               User.name, Article.title). \
    join(User, ChangeLogs.author_id == User.id). \
    join(Article, Article.id == ChangeLogs.article_id). \
    filter(Article.id == 1, User.id == 1).all()
	
	# 子查询关联一，查询文章最新修改的内容
	# 定义子查询
	from sqlalchemy import and_
	
	stmt =db.session.query(ChangeLogs.article_id, ChangeLogs.modify_content,
						   ChangeLogs.create_time ). \
        filter(ChangeLogs.article_id == Article.id,
               ChangeLogs.create_time == Article.last_change_time
               ).subquery()
    # 连接子查询内容
	last_change_logs = db.session.query(Article.title,stmt.c.modify_content,
									Article.last_change_time
									).\
		outerjoin(stmt, and_(stmt.c.article_id== Article.id,
	                         stmt.c.create_time == Article.last_change_time
	                         )).all()
	
	# 子查询关联二,查询所有作者的文章数
	# 查询文章数
	article_count = db.session.query(db.func.count(Article.id)). \
	    filter(Article.user_id == User.id).correlate(User).as_scalar()
	user_article = db.session.query(User.name, article_count.label("count")). \
	    all()
	```
5. 排序
	```python
	# oder_by  asc正序 desc 倒叙
	# 查询某个作者的所有文章并按照最后修改时间倒叙排列
	user_articles= db.session.query(Article).\
	join(User,Article.author_id == User.id).filter(User.id == 1).\
	order_by(Article.last_change_time.desc()).all()
	```
6. 分组
	```python
	# group_by，统计不同年龄段的用户
	group_age = db.session.query(User.age, db.func.count(User.id)). \
    group_by(User.age).all()
	```
7. 函数 [相关文档](https://docs.sqlalchemy.org/en/latest/core/functions.html?highlight=func#module-sqlalchemy.sql.functions)
	```python
	# 简单的介绍几种函数，更多的可以看官方文档
	# count 统计不同年龄段的用户
	group_age = db.session.query(User.age, db.func.count(User.id)). \
    group_by(User.age).all()
    # sum 总年龄
    sum_age= db.session.query(db.func.sum(User.age)).first()
    # max 最大年龄
    max_age = db.session.query(db.func.max(User.age)).first()
    # min 最新年龄 
    min_age = db.session.query(db.func.min(User.age)).first()
    # avg 平均年龄
    avg_age = db.session.query(db.func.avg (User.age)).first()
	```
 
## 改
1. 单个修改(更新)
	```python
	# 方法一
	user =db.session.query(User).fitler(User.id == 1).first()
	user.age = 25
	user.name ="qinzq"
	db.session.add(user) 
	db.session.commit()
	# 方法二
	db.session.query(User).filter(User.id == 1).\
	update({"age ":25,"name":"qinzq"})
	```
2. 批量修改(更新)
	```python
	db.session.query(User).filter(User.id.in_((1, 2, 3))).update({"age ":25})
	db.session.commit()
	```
 
## 删
1. 单个删除
	```python
	user =db.session.query(User).fitler(User.id == 1).first()
	db.session.delete(user) 
	db.session.commit()
	```
2. 批量删除
	```python
	db.session.query(User).filter(User.id.in_((1, 2, 3))).\
	delete(synchronize_session=False)
	db.session.commit() 
	```
3. 删除整个表数据
	```python
	db.session.query(User).delete(synchronize_session=False)
	db.session.commit() 
	```
 

## 事务
1. rollback回滚错误操作
	```python
		new_user = User()
		new_user.name = 'qin'
		new_user.age =23
		try:
			db.session.add(new_user)
			db.session.commit()
		except:
			db.session.rollback()
	```
2. flush预提交数据
	```python
	new_user_qin = User(name = 'qin',age =23)
	db.session.add(new_user_qin)
	db.session.flush()
	
	new_user_zhang = User(name='zhang',age=26)
	db.session.add(new_user_zhang )
	db.session.flush()
	
	# 统一执行commit操作，如果发生错误回滚添加的两个用户
	try:
		db.session.commit()
	except:
		db.session.rollback()
	```
 
## 总结
- 本文简单的介绍了Flask与SQLAlchemy集成方法以及SQLAlchemy的简单使用
- 读者想了解更多可以去[SQLAlchemy官方文档](https://www.sqlalchemy.org)详细了解
- 下一篇文章将实现基于Flask与SQLAlchemy的单表接口