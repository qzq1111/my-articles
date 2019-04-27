# Flask-APScheduler定时任务

tips:
- 本文基于python3编写
- [代码仓库](https://github.com/qzq1111/flask-resful-example)

## 项目背景
在项目中会遇到各种定时任务，比如定时清理文件，定时计算报表等。

## APScheduler
APScheduler的全称是Advanced Python Scheduler。它是一个轻量级的 Python 定时任务调度框架。APScheduler 支持三种调度任务：固定时间间隔，固定时间点（日期），Linux 下的 Crontab 命令。同时，它还支持异步执行、后台执行调度任务。[官方文档](https://apscheduler.readthedocs.io/en/latest/)
### 一、简单使用
1. 安装`pip install apscheduler`
2. 示例，每5秒输出时间
    ```python
    from apscheduler.schedulers.blocking import BlockingScheduler
    from datetime import datetime

    def timed_task():
        print(datetime.now().strftime("%Y-%m-%d %H:%M:%S"))

    if __name__ == '__main__':
        # 创建当前线程执行的schedulers
        scheduler = BlockingScheduler()
        # 添加调度任务(timed_task),触发器选择interval(间隔性),间隔时长为5秒
        scheduler.add_job(timed_task, 'interval', seconds=5)
        # 启动调度任务
        scheduler.start()
    ```
### 二、调度器（scheduler）
- `BlockingScheduler`: 调度器在当前进程的主线程中运行，会阻塞当前线程。
- `BackgroundScheduler`: 调度器在后台线程中运行，不会阻塞当前线程。
- `AsyncIOScheduler`: 结合asyncio模块一起使用。
- `GeventScheduler`: 程序中使用gevent作为IO模型和GeventExecutor配合使用。
- `TornadoScheduler`: 程序中使用Tornado的IO模型，用 ioloop.add_timeout 完成定时唤醒。
- `TwistedScheduler`: 配合TwistedExecutor，用reactor.callLater完成定时唤醒。
- `QtScheduler`: 应用是一个Qt应用，需使用QTimer完成定时唤醒。
### 三、触发器（trigger）
- `date`是最基本的一种调度，作业任务只会执行一次。[参数详见](https://apscheduler.readthedocs.io/en/latest/modules/triggers/date.html#module-apscheduler.triggers.date)

- `interval`触发器，固定时间间隔触发。[参数详见](https://apscheduler.readthedocs.io/en/latest/modules/triggers/interval.html#module-apscheduler.triggers.interval)

- `cron` 触发器，在特定时间周期性地触发，和Linux crontab格式兼容。它是功能最强大的触发器。[参数详见](https://apscheduler.readthedocs.io/en/latest/modules/triggers/cron.html#module-apscheduler.triggers.cron)

### 四、作业存储（job store）
- 添加任务，有两种添加方法，一种`add_job()`， 另一种是`scheduled_job()`修饰器来修饰函数。
    ```python
    from datetime import datetime
    from apscheduler.schedulers.blocking import BlockingScheduler
    scheduler = BlockingScheduler()

    # 第一种
    @scheduler.scheduled_job(job_func, 'interval', seconds=10)
    def timed_task():
        print(datetime.now().strftime("%Y-%m-%d %H:%M:%S"))
    # 第二种
    scheduler.add_job(timed_task, 'interval', seconds=5)

    scheduler.start()
    ```
- 删除任务，两种方法：`remove_job()` 和 `job.remove()`。`remove_job()`是根据任务的id来移除，所以要在任务创建的时候指定一个 id。`job.remove()`则是对任务执行remove方法。
    ```python
    scheduler.add_job(job_func, 'interval', seconds=20, id='one')
    scheduler.remove_job(one)

    task = add_job(task_func, 'interval', seconds=2, id='job_one')
    task.remvoe()
    ```

- 获取任务列表，通过`scheduler.get_jobs()`方法能够获取当前调度器中的所有任务的列表
    ```python
   tasks = scheduler.get_jobs()
    ```
- 关闭任务，使用`scheduler.shutdown()`默认情况下调度器会等待所有正在运行的作业完成后，关闭所有的调度器和作业存储。
    ```python
    scheduler.shutdown()
    scheduler.shutdown(wait=false)
    ```

### 五、执行器（executor）
执行器是执行调度任务的模块。最常用的 executor 有两种：`ProcessPoolExecutor` 和 `ThreadPoolExecutor`

## Flask与APScheduler结合
1. 安装`pip install flask_apscheduler`
2. 将apscheduler注册到Flask App
    - 编写 `core.py`
        ```python
        from flask_apscheduler import APScheduler
        scheduler = APScheduler()
        ```
    - 编写`factory.py`
        ```python
        from core import scheduler
        def create_app():
            app = Flask(__name__)
            # 配置任务，不然无法启动任务
            app.config.update(
                {"SCHEDULER_API_ENABLED": True,
                 "JOBS": [{"id": "my_job", # 任务ID
                            "func": "task:my_job",#任务位置
                            "trigger": "interval", #触发器
                            "seconds": 5 # 时间间隔
                            } 
                        ]}
                )
            scheduler.init_app(app)
            scheduler.start()
            return app
        ```
3. 编写定时任务`task.py`
    ```python
    def my_job():
        print(datetime.now().strftime('%Y-%m-%d %H:%M:%S'))
    ```
4. 创建`app.py`
    ```python
    from factory import create_app

    app = create_app()
    if __name__ == "__main__":
        app.run()
    ```
5. 测试 执行`python app.py`，控制台输出如下：
    ```shell
    Running job "my_job (trigger: interval[0:00:05], next run at: 2019-04-27 17:26:31 CST)" (scheduled at 2019-04-27 17:26:31.477195+08:00)
    Job "my_job (trigger: interval[0:00:05], next run at: 2019-04-27 17:26:36 CST)" executed successfully
    2019-04-27 17:26:31
    Running job "my_job (trigger: interval[0:00:05], next run at: 2019-04-27 17:26:33 CST)" (scheduled at 2019-04-27 17:26:33.410837+08:00)
    Job "my_job (trigger: interval[0:00:05], next run at: 2019-04-27 17:26:38 CST)" executed successfully
    2019-04-27 17:26:33
    ```
 
##  踩坑点
1. 多进程部署，定时任务重复启动解决方法
    - 解决思路：在启动任务时，设置文件锁，当能获取到文件锁时，不在启动任务
    - 代码
        ```python
        #factory.py
        def create_app():
            app =Flask(__name__)
            # 启动定时任务
            scheduler_init(app)
            return app

        def scheduler_init(app):
            """
            保证系统只启动一次定时任务
            :param app:
            :return:
            """
            if platform.system() != 'Windows':
                fcntl = __import__("fcntl")
                f = open('scheduler.lock', 'wb')
                try:
                    fcntl.flock(f, fcntl.LOCK_EX | fcntl.LOCK_NB)
                    scheduler.init_app(app)
                    scheduler.start()
                    app.logger.debug('Scheduler Started,---------------')
                except:
                    pass

                def unlock():
                    fcntl.flock(f, fcntl.LOCK_UN)
                    f.close()

                atexit.register(unlock)
            else:
                msvcrt = __import__('msvcrt')
                f = open('scheduler.lock', 'wb')
                try:
                    msvcrt.locking(f.fileno(), msvcrt.LK_NBLCK, 1)
                    scheduler.init_app(app)
                    scheduler.start()
                    app.logger.debug('Scheduler Started,----------------')
                except:
                    pass

                def _unlock_file():
                    try:
                        f.seek(0)
                        msvcrt.locking(f.fileno(), msvcrt.LK_UNLCK, 1)
                    except:
                        pass

                atexit.register(_unlock_file)
        ```

2. Gunicorn使用gevent模式无效解决方法
    - 解决思路：将gunicorn启动模式换为`eventlet`
    - 配置文件`gun.conf`
        ```
        # 并行工作进程数
        workers = 4
        # 指定每个工作者的线程数
        threads = 4
        # 监听内网端口80
        bind = '0.0.0.0:80'
        # 工作模式协程
        worker_class = 'eventlet' 
        # 设置最大并发量
        worker_connections = 2000
        # 设置进程文件目录
        pidfile = 'gunicorn.pid'
        # 设置访问日志和错误信息日志路径
        accesslog = './logs/gunicorn_acess.log'
        errorlog = './logs/gunicorn_error.log'
        # 设置日志记录水平
        loglevel = 'info'
        # 代码发生变化是否自动重启
        reload=True
        ```


3. 使用Flask数据库解决方法
    - 解决思路：数据注册时指定一下app，并在定时任务中使用数据库绑定的app栈
    - `factory.py`配置
        ```python
        def create_app():
            app =Flask(__name__)
            # 数据库注册
            db.app = app
            db.init_app(app)
            app.config.update(
                {"SCHEDULER_API_ENABLED": True,
                 "JOBS": [{"id": "db_query", # 任务ID
                            "func": "task:db_query",#任务位置
                            "trigger": "interval", #触发器
                            "seconds": 5 # 时间间隔
                            } 
                        ]}
                )
            scheduler_init(app)
            return app
        ```

    - `task.py`中使用数据库
        ```python
        from app.utils.core import db

        def db_query():
            """
            定时任务使用数据库
            """
            with db.app.app_context():
                 data =db.session.query(user).frist()
                 print(data)
        
        ```

## 总结
- 简单的使用了Flask-APScheduler
- 实际生产中遇到的问题解决方法
- 下一章将介绍Flask实现图形验证码及验证 