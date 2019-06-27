# Flask后端实践  连载十九 Flask工厂模式集成使用Celery

tips:
- 讲解Flask与Celery结合使用中遇到的各种问题解决方法
- 本文基于python3编写
- [代码仓库](https://github.com/qzq1111/flask-resful-example)

## 项目场景

项目上有许多任务需要在后台处理，虽然可以使用异步线程来解决，但是无法及时获取到任务执行状态，有时任务执行失败，也无法及时获取到关键信息。
因此，采用Celery来重写相关的异步任务，方便管理和处理错误信息。

## 前置条件

- 安装Celery `pip install celery==4.3.0`
- 安装Flask  `pip install flask==1.0.3`
- 安装redis  `pip install redis==3.2.1`


## Flask与Celery结合简单使用
- 新建`celery_app.py`
    ```python
    from celery import Celery
    
    redis_config='redis://127.0.0.1:6379/0'
    celery_app = Celery(__name__, backend=redis_config, broker=redis_config)
    
    
    @celery_app.task
    def async_add(x, y):
        return x + y

    ```
- 新建`flask_app.py`
    ```python
    from flask import Flask
    from celery_app import async_add
    
    flask_app = Flask(__name__)
    @flask_app.route('/test', methods=["GET"])
    def test_add():
        """
        测试相加
        :return:
        """
        result = async_add.delay(1, 2)
        return str(result.get(timeout=1))
    
    if __name__ == '__main__':
        flask_app.run()

    ```
- 启动
    - windows shell中`celery -A celery_app worker --pool=solo -l info`
    - Linux shell中` celery -A celery_app worker -l info`
    - 启动Flask APP `python flask_app.py`
- 测试
    - 访问`http://127.0.0.1:5000/test` 返回 `3`
    
## 工厂模式下Flask与Celery结合
- 文件目录结构
    ```
    _
     |_ app
     |  |_ __init__.py # 工程函数
     |  |_ api.py  # API接口
     |  |_ celery.py # celery及tasks
     |  
     |_ run.py # 启动Flask APP
    
    ```

- 编写`celery.py`
    ```python
    from celery import Celery
    from flask import current_app
    
    celery_app = Celery(__name__)
    
    
    @celery_app.task
    def add(x, y):
        """
        加法
        :param x:
        :param y:
        :return:
        """
        return str(x + y)
    
    
    @celery_app.task
    def flask_app_context():
        """
        celery使用Flask上下文
        :return:
        """
        with current_app.app_context():
            return str(current_app.config)

    ```

- 编写`__init__.py`
    ```python
    from flask import Flask
    from .celery import celery_app
    
    
    def create_app():
        app = Flask(__name__)
    
        celery_app.conf.update({"broker_url": 'redis://127.0.0.1:6379/0',
                                "result_backend": 'redis://127.0.0.1:6379/0', })
        from .api import bp
    
        app.register_blueprint(bp)
        return app

    ```
- 编写`api.py`
    ```python
    from flask import Blueprint
    from .celery import add, flask_app_context
    
    bp = Blueprint("test", __name__, url_prefix='/')
    
    
    @bp.route('/testAdd', methods=["GET"])
    def test_add():
        """
        测试相加
        :return:
        """
        result = add.delay(1, 2)
        return result.get(timeout=1)
    
    
    @bp.route('/testFlaskAppContext', methods=["GET"])
    def test_flask_app_context():
        """
        测试相加
        :return:
        """
        result = flask_app_context.delay()
        return result.get(timeout=1).replace('<Config', '')
    ```
- 编写`run.py`
    ```python
    from app import create_app,celery_app

    app = create_app()
    # 关键点，往celery推入flask信息，使得celery能使用flask上下文
    app.app_context().push()
    
    
    if __name__ == '__main__':
        app.run()


    ```

- 启动
  - windows shell中输入`celery -A run:celery_app worker --pool=solo -l info`
  - Linux shell中输入`celery -A run:celery_app worker -l info`
  - 启动Flask APP `python run.py`
- 测试
  - 访问`http://127.0.0.1:5000/testAdd` 返回 `3`
  - 访问`http://127.0.0.1:5000/testFlaskAppContext` 返回 Flask配置信息

## 总结
- 本片文章主要解决了新版本的Celery使用Flask上下文的功能
