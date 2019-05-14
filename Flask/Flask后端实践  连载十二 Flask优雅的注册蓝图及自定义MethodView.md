# Flask后端实践  连载十二 Flask优雅的注册蓝图及自定义MethodView
tips:
- 介绍如何有效的统一注册蓝图及自定义MethodView
- 本文基于python3编写
- [代码仓库](https://github.com/qzq1111/flask-resful-example)

## 目的
在项目开发中，通常采用工程模式来创建app，如果注册接口写在工厂函数中，不仅不好管理，而且代码也看起来很臃肿。并且项目中会有很多模块，每个模块又有不同的功能，由此分割出来的接口也非常多。所有需要有统一的地方来管理接口及相关模块。
## 解决方法
1. 编写蓝图(`user.py`)
    ```python
    from flask import Blueprint

    bp = Blueprint("test", __name__, url_prefix='/')


    @bp.route('/testBP', methods=["GET"])
    def test_bp():
        return "蓝图测试"

    ```
2. 编写自定义MethodView(`auth.py`)
    ```python
    from flask.views import MethodView

    class AuthMethodView(MethodView):
        # 指定需要启用的请求方法
        __methods__ = ["GET", "POST", "PUT"]

        def get(self):
            return "测试自定义MethodView"

        def post(self):
            return "测试自定义MethodView"

        def put(self):
            return "测试自定义MethodView"

        def delete(self):
            return "测试自定义MethodView"    
    ```
3. 统一管理蓝图和自定义MethodView(`router.py`)
    ```python
    from user import bp as user_bp
    from auth import AuthMethodView

    router = [
        user_bp, # 用户蓝图接口
        AuthMethodView, # 权限自定义MethodView
    ]
    ```

4. 统一注册蓝图和自定义MethodView(`app.py`)
    ```python
    from flask import Flask, Blueprint
    from router import router


    def create_app():
        """
        工厂模式创建APP
        """
        app = Flask(__name__)
        # 注册接口
        register_api(app, router)

        return app


    def register_api(app, routers):
        """
        注册蓝图和自定义MethodView
        """
        for router in routers:
            if isinstance(router, Blueprint):
                app.register_blueprint(router)
            else:
                try:
                    endpoint = router.__name__
                    view_func = router.as_view(endpoint)
                    # url默认为类名小写
                    url = '/{}/'.format(router.__name__.lower())
                    if 'GET' in router.__methods__:
                        app.add_url_rule(url, defaults={'key': None}, view_func=view_func, methods=['GET', ])
                        app.add_url_rule('{}<string:key>'.format(url), view_func=view_func, methods=['GET', ])
                    if 'POST' in router.__methods__:
                        app.add_url_rule(url, view_func=view_func, methods=['POST', ])
                    if 'PUT' in router.__methods__:
                        app.add_url_rule('{}<string:key>'.format(url), view_func=view_func, methods=['PUT', ])
                    if 'DELETE' in router.__methods__:
                        app.add_url_rule('{}<string:key>'.format(url), view_func=view_func, methods=['DELETE', ])
                except Exception as e:
                    raise ValueError(e)


    if __name__ == '__main__':
        app = create_app()
        app.run()

    ```
5. 测试，启动app
- 浏览器访问`http://127.0.0.1:5000/testBP`，页面返回`蓝图测试`

- 浏览器访问`http://127.0.0.1:5000/authmethodview/` ，页面返回`测试自定义MethodView`


## 总结
- 统一管理了蓝图和自定义的MethodView，并减少了代码的冗余，保持代码的整洁性。
- 下一篇将介绍使用flask输出excel报表