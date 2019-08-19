# Flask后端实践  连载二十 Flask接口单元测试+测试报告+测试覆盖率

tips:
- flask接口在工程中的单元测试、测试报告、测试覆盖率
- 本文基于python3编写
- [代码仓库](https://github.com/qzq1111/flask-resful-example)

## 前言

不管喜不喜欢写测试代码，终究自己的应用程序都会被测试，自己应用程序的用户将成为测试者。在用户使用过程测试出现问题，往往都需要自己顶着压力去修改，那为何不早早将测试做好。

## Flask使用unittest测试
1. 编写Flask接口`app.py`

    ```python
    from flask import Flask, request, jsonify

    app = Flask(__name__)


    @app.route('/add', methods=["GET"])
    def add():
        x = request.args.get("x", type=float)
        y = request.args.get("y", type=float)
        if x is None or y is None:
            return jsonify("err")
        return jsonify(x + y)


    if __name__ == '__main__':
        app.run()
    ```
2. 编写测试代码`test_app.py`

    ```python
    from app import app
    import unittest


    class TestAdd(unittest.TestCase):

        def setUp(self) -> None:
            app.config['TESTING'] = True # 开启测试环境
            self.app = app.test_client()  
            
        def tearDown(self) -> None:
            pass

        def test_add_no_parameters(self):
            """
            未传参数
            :return:
            """
            rp = self.app.get("/add")
            data = rp.json
            self.assertEqual(data, "err")

        def test_add_x(self):
            """
            传入x
            :return:
            """
            rp = self.app.get("/add", query_string={"x": 1, })
            data = rp.json
            self.assertEqual(data, 'err')

        def test_add_y(self):
            """
            传入y
            :return:
            """
            rp = self.app.get("/add", query_string={"y": 1, })
            data = rp.json
            self.assertEqual(data, 'err')

        def test_add_strings(self):
            """
            传入值类型错误
            :return:
            """
            rp = self.app.get("/add", query_string={"y": "x", "x": "y"})
            data = rp.json
            self.assertEqual(data, 'err')

        def test_add_ok(self):
            """
            传入值正确
            :return:
            """
            rp = self.app.get("/add", query_string={"y": 1, "x": 1})
            data = rp.json
            self.assertEqual(data, 2)


    if __name__ == '__main__':
        unittest.main(verbosity=2)

    ```
    - 命令行运行`python test_app.py`
    - 查看未通过的测试用例，并修复错误
## Flask工程模式测试
1. 项目目录

    ```
    _
     |_ app
     |  |_ __init__.py # 工程函数
     |  |_ api.py  # API接口
     |
     |_ tests
     |  |_ __init__.py
     |  |_ HTMLTestRunner.py
     |  |_ test_add.py
     |  |_ test_subtract.py
     |  |_ run_test.py
     | 
     |_ run.py # 启动Flask APP
    
    ```

2. 编写工程函数`__init__.py`

    ```python
    from flask import Flask

    def create_app():
        app = Flask(__name__)

        from .api import bp
        app.register_blueprint(bp)

        return app
    ```

3. 编写API接口`api.py`

    ```python
    from flask import Blueprint, request, jsonify

    bp = Blueprint("api", __name__, url_prefix='/')


    @bp.route('/add', methods=["GET"])
    def add():
        x = request.args.get("x", type=float)
        y = request.args.get("y", type=float)
        if x is None or y is None:
            return jsonify("err")
        return jsonify(x + y)
    
    @bp.route('/subtract', methods=["GET"])
    def subtract():
        x = request.args.get("x", type=float)
        y = request.args.get("y", type=float)
        if x is None or y is None:
            return jsonify("err")
        return jsonify(x - y)
    ```

4. 编写测试代码`test_add.py`和`test_subtract.py`

    - `test_add.py`测试加法
    
    ```python
    # test_add.py
    import unittest
    from app import create_app


    class TestAdd(unittest.TestCase):

        def setUp(self) -> None:
            app = create_app()
            app.config['TESTING'] = True
            self.app = app.test_client()

        def test_add_no_parameters(self):
            """
            未传参数
            :return:
            """
            rp = self.app.get("/add")
            data = rp.json
            self.assertEqual(data, "err")

        def test_add_x(self):
            """
            传入x
            :return:
            """
            rp = self.app.get("/add", query_string={"x": 1, })
            data = rp.json
            self.assertEqual(data, 'err')

        def test_add_y(self):
            """
            传入y
            :return:
            """
            rp = self.app.get("/add", query_string={"y": 1, })
            data = rp.json
            self.assertEqual(data, 'err')

        def test_add_strings(self):
            """
            传入值类型错误
            :return:
            """
            rp = self.app.get("/add", query_string={"y": "x", "x": "y"})
            data = rp.json
            self.assertEqual(data, 'err')

        def test_add_ok(self):
            """
            传入值正确
            :return:
            """
            rp = self.app.get("/add", query_string={"y": 1, "x": 1})
            data = rp.json
            self.assertEqual(data, 2)


    if __name__ == '__main__':
        unittest.main(verbosity=2)

    ```

    - `test_subtract.py`测试减法
    
    ```python
    import unittest
    from app import create_app


    class TestSubtract(unittest.TestCase):

        def setUp(self) -> None:
            app = create_app()
            app.config['TESTING'] = True
            self.app = app.test_client()

        def test_subtract_no_parameters(self):
            """
            未传参数
            :return:
            """
            rp = self.app.get("/subtract")
            data = rp.json
            self.assertEqual(data, "err")

        def test_subtract_x(self):
            """
            传入x
            :return:
            """
            rp = self.app.get("/subtract", query_string={"x": 1, })
            data = rp.json
            self.assertEqual(data, 'err')

        def test_subtract_y(self):
            """
            传入y
            :return:
            """
            rp = self.app.get("/subtract", query_string={"y": 1, })
            data = rp.json
            self.assertEqual(data, 'err')

        def test_subtract_strings(self):
            """
            传入值类型错误
            :return:
            """
            rp = self.app.get("/subtract", query_string={"y": "x", "x": "y"})
            data = rp.json
            self.assertEqual(data, 'err')

        def test_subtract_ok(self):
            """
            传入值正确
            :return:
            """
            rp = self.app.get("/subtract", query_string={"y": 1, "x": 1})
            data = rp.json
            self.assertEqual(data, 1)


    if __name__ == '__main__':
        unittest.main(verbosity=2)



    ```
5. 测试
   - 命令行运行`python test_add.py`或`python test_subtract.py`
   - 查看未通过的测试用例，并修复错误
## Flask测试报告
1. 当有多个测试用例时，采用python的命令不方便，unittest中可以加载指定路径下符合条件的测试用例。并且可以生成Text报告`run_test.py`
   
    ```python
    import unittest

    test_dir = './'
    discover = unittest.defaultTestLoader.discover(test_dir, pattern='*test_*.py')

    if __name__ == '__main__':
        with open('UnittestTextReport.txt', 'a') as f:
            runner = unittest.TextTestRunner(stream=f, verbosity=2)
            runner.run(discover)

    ```
2. 当然也可以生成HTML报告，可以使用[HTMLTestRunner](http://tungwaiyip.info/software/HTMLTestRunner.html)，不过该包已经很久没有更新了，只能在python2的环境下使用，参考[HTMLTestRunner修改成Python3版本](https://www.cnblogs.com/sgtb/p/4169732.html)修改之后python3环境下使用。
    
    ```python
    import unittest
    from tests.HTMLTestRunner import HTMLTestRunner

    test_dir = './'
    discover = unittest.defaultTestLoader.discover(test_dir, pattern='*test_*.py')

    if __name__ == '__main__':
        with open('HtmlReport.html', 'wb') as f:
            runner = HTMLTestRunner(stream=f, title='test report', description='', verbosity=2)
            runner.run(discover)
    ```


## Flask测试覆盖率
1. 安装coverage，`pip install coverage`

2. coverage命令

    coverage命令格式`coverage <command> [options] [args]`

    |命令|含义|
    |---|---|
    |annotate |运行一个python程序并收集运行数据|
    |combine  |合并覆盖报告|
    |erase    |删除之前收集的统计数据|
    |help     |帮助|
    |html     |创建Html报告|
    |report   |覆盖率统计数据报告|
    |run      |运行Python程序并测量代码执行。|
    |xml      |创建Xml报告|


3. coverage覆盖报告

   - 命令行进入到项目根目录
   - 命令行输入`coverage run --source='app'  -m tests.run_test`测试覆盖
   - 命令行输入`coverage report`

      ```shell
      Name              Stmts   Miss  Cover
      -------------------------------------
      app\__init__.py       6      0   100%
      app\api.py           14      0   100%
      -------------------------------------
      TOTAL                20      0   100%
      ```
   - 命令行输入`coverage html`，在当前目录生成文件夹**htmlcov**，打开文件夹中的**index.html**文件查看覆盖报告

## 总结
- 使用unittest测试Flask接口
- 生成测试报告和测试覆盖报告