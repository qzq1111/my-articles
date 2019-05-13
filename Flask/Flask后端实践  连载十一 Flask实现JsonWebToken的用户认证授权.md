# Flask后端实践  连载十一 Flask实现JsonWebToken的用户认证授权
tips：
- 本文实现JsonWebToken的用户认证授权
-   本文基于python3编写
- [代码仓库](https://github.com/qzq1111/flask-resful-example)
## 项目场景
由于公司项目都是前后端分离，需要处理用户认证方面的问题，以及方便应用的扩展。便采用了JWT的方式。
## JWT
### JWT认证流程
- 用户发送登陆请求到服务端
- 服务端验证用户的信息
- 服务端通过验证发送给用户数据访问token和刷新token
- 客户端存储token，并在每次请求时附送上这个token值
- 服务端验证token值，并返回数据
- 当服务端验证token失败，客户端使用刷新token刷新数据访问token，并重新请求数据。
### JWT构成
JWT一共由三部分组成，header（头部）、payload（载荷）、signature（签名）。
1. header，一共两部分，最后转base64。
   - 类型
   - 加密算法
    ```json
    {
        "type":"JWT",
        "alg":"HS256"
    }
    ```
2. payload，下面的参数建议都填写但不是强制使用，也可以自定义载荷，不可以存入敏感信息，最后转base64。
   - iss: jwt签发者
   - sub: jwt所面向的用户
   - aud: 接收jwt的一方
   - exp: jwt的过期时间，这个过期时间必须要大于签发时间
   - nbf: 定义在什么时间之前，该jwt都是不可用的.
   - iat: jwt的签发时间
   - jti: jwt的唯一身份标识，主要用来作为一次性token,从而回避重放攻击。
   ```json
   {
       "iss":"qin",
       "exp":1557715124,
       "user_name":"zhang"
   }
   ```
3. signature，一共三部分。转base64的header和转base64的payload拼接之后，然后使用header中声明的加密方式和secret加盐的方式加密字符串。
   - 转base64的header
   - 转base64的payload
   - secret（私钥）
4. 生成的token如下所示
    ```
    eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NTc3MjI1NTgsImlzcyI6InFpbiIsInVzZXJfbmFtZSI6InpoYW5nIn0.YHNkSdAMEUIY__U5f9e1tQFAdqiHv_ai_gfaPpPnWLc
    ```
## Flask JWT结合
1. 安装PyJWT `pip install PyJWT`
2. 编写刷新token和数据请求token生成函数和解密函数(`util.py`)
    ```python
    from datetime import datetime, timedelta

    import jwt

    key = "zkpfw*%$qjrfono@sdko34@%"


    def generate_access_token(user_name: str = "", algorithm: str = 'HS256', exp: float = 2):
        """
        生成access_token
        :param user_name: 自定义部分
        :param algorithm:加密算法
        :param exp:过期时间
        :return:token
        """

        now = datetime.utcnow()
        exp_datetime = now + timedelta(hours=exp)
        access_payload = {
            'exp': exp_datetime,
            'flag': 0,  # 标识是否为一次性token，0是，1不是
            'iat': now,  # 开始时间
            'iss': 'qin',  # 签名
            'user_name': user_name  # 自定义部分
        }
        access_token = jwt.encode(access_payload, key, algorithm=algorithm)
        return access_token


    def generate_refresh_token(user_name: str = "", algorithm: str = 'HS256', fresh: float = 30):
        """
        生成refresh_token

        :param user_name: 自定义部分
        :param algorithm:加密算法
        :param fresh:过期时间
        :return:token
        """
        now = datetime.utcnow()
        # 刷新时间为30天
        exp_datetime = now + timedelta(days=fresh)
        refresh_payload = {
            'exp': exp_datetime,
            'flag': 1,  # 标识是否为一次性token，0是，1不是
            'iat': now,  # 开始时间
            'iss': 'qin',  # 签名，
            'user_name': user_name  # 自定义部分
        }

        refresh_token = jwt.encode(refresh_payload, key, algorithm=algorithm)
        return refresh_token

    def decode_auth_token(token: str):
        """
        解密token
        :param token:token字符串
        :return:
        """
        try:
            # 取消过期时间验证
            # payload = jwt.decode(token, key, options={'verify_exp': False})
            payload = jwt.decode(token, key=key, )
        except (jwt.ExpiredSignatureError, jwt.InvalidTokenError, jwt.InvalidSignatureError):
            return ""
        else:
            return payload

    def identify(auth_header: str):
        """
        用户鉴权
        :return: 
        """
        if auth_header:
            payload = decode_auth_token(auth_header)
            if not payload:
                return False
            if "user_name" in payload and "flag" in payload:
                if payload["flag"] == 1:
                    # 用来获取新access_token的refresh_token无法获取数据
                    return False
                elif payload["flag"] == 0:
                    return payload["user_name"]
                else:
                    # 其他状态暂不允许
                    return False
            else:
                return False
        else:
            return False
    ```
3. 编写登陆保护函数(`util.py`)
    ```python
    from functools import wraps

    def login_required(f):
        """
        登陆保护，验证用户是否登陆
        :param f:
        :return:
        """

        @wraps(f)
        def wrapper(*args, **kwargs):
            token = request.headers.get("Authorization", default=None)
            if not token:
                return "请登陆"
            user_name = identify(token)
            if not user_name:
                 return "请登陆"
            # 获取到用户并写入到session中,方便后续使用
            session["user_name"] = user_name  
            return f(*args, **kwargs)
        return wrapper
    ```
4. 编写接口(app.py)
    ```python
    from flask import Flask
    from util import *

    app = Flask(__name__)
    app.config["SECRET_KEY"] = "reqweqwcasd!#$%456421&^%&^%"


    @app.route('/testLogin', methods=["POST"])
    def test_login():
        """
        登陆成功获取到数据获取token和刷新token
        :return:
        """
        obj = request.get_json(force=True)
        name = obj.get("name")
        if not obj or not name:
            return "参数错误"

        if name == "qin":
            access_token = generate_access_token(user_name=name)
            refresh_token = generate_refresh_token(user_name=name)
            data = {"access_token": access_token.decode("utf-8"), 
            "refresh_token": refresh_token.decode("utf-8")}
            return jsonify(data)
        else:
            return "用户名或密码错误"


    @app.route('/testGetData', methods=["GET"])
    @login_required
    def test_get_data():
        """
        测试登陆保护下获取数据
        :return:
        """
        name = session.get("user_name")

        return "{}，你好！！".format(name)


    @app.route('/testRefreshToken', methods=["GET"])
    def test_refresh_token():
        """
        刷新token，获取新的数据获取token
        :return:
        """
        refresh_token = request.args.get("refresh_token")
        if not refresh_token:
            return "参数错误"
        payload = decode_auth_token(refresh_token)
        if not payload:
            return "请登陆"
        if "user_name" not in payload:
            return "请登陆"
        access_token = generate_access_token(user_name=payload["user_name"])
        data = {"access_token": access_token.decode("utf-8"), "refresh_token": refresh_token}
        return jsonify(data)


    if __name__ == '__main__':
        app.run()
    ```
 
## 测试
1. 测试登陆
    ```
    请求链接：http://127.0.0.1:5000/testLogin
    请求方式：POST
    请求数据：{"name":"qin"}
    数据方式：json

    服务端响应:
    {"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NTc3MzM5OTUsImZsYWciOjAsImlhdCI6MTU1NzcyNjc5NSwiaXNzIjoicWluIiwidXNlcl9uYW1lIjoicWluIn0.PBWk8LOB_S4TVRg7BrXQ9vGjjM31veqgkgbyinVdlVc",
    "refresh_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NjAzMTg3OTUsImZsYWciOjEsImlhdCI6MTU1NzcyNjc5NSwiaXNzIjoicWluIiwidXNlcl9uYW1lIjoicWluIn0.kn0-TkP79XlUbCZDeCX7R6oFvG9-M1kYER_7P_d0dTM"}
    ```
2. 测试获取数据
    ```
    请求链接：http://127.0.0.1:5000/testGetData
    请求方式：GET
    请求头：Authorization = eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NTc3MzM5OTUsImZsYWciOjAsImlhdCI6MTU1NzcyNjc5NSwiaXNzIjoicWluIiwidXNlcl9uYW1lIjoicWluIn0.PBWk8LOB_S4TVRg7BrXQ9vGjjM31veqgkgbyinVdlVc

    服务端响应:
    qin，你好！！
    ```
3. 测试刷新token
    ```
    请求链接：http://127.0.0.1:5000/testRefreshToken
    请求方式：GET
    请求参数：refresh_token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NjAzMTg3OTUsImZsYWciOjEsImlhdCI6MTU1NzcyNjc5NSwiaXNzIjoicWluIiwidXNlcl9uYW1lIjoicWluIn0.kn0-TkP79XlUbCZDeCX7R6oFvG9-M1kYER_7P_d0dTM

    服务端响应:{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NTc3MzQwNjksImZsYWciOjAsImlhdCI6MTU1NzcyNjg2OSwiaXNzIjoicWluIiwidXNlcl9uYW1lIjoicWluIn0.z-DLcBRRh6pE_wQqfF_YjQMxupbGVI2KD-v9jzEz-H0","refresh_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NjAzMTg3OTUsImZsYWciOjEsImlhdCI6MTU1NzcyNjc5NSwiaXNzIjoicWluIiwidXNlcl9uYW1lIjoicWluIn0.kn0-TkP79XlUbCZDeCX7R6oFvG9-M1kYER_7P_d0dTM"
    }
    ```
## 思考
前文只是简单现实了flask和jwt的结合，实际中使用也会有一定的问题。
- 第一：载荷可以直接用base64解密。
- 第二：如果截获了token，可以利用暴力破解等方式，直接破解加密，提升用户权限。
- 第三：一旦拿到刷新token，就可以无限次获取授权，直到刷新token过期。

一些解决思路：
- 载荷不存入敏感信息
- 用户进行某项关键操作，再次验证用户。
- 限制请求次数
- 服务端管理token有效性
## 总结
- 本文介绍了Flask和JWT的结合，以及实际使用中的一些问题的解决办法。
- 下一篇文章将讲解一下如果优雅的注册蓝图和自定义的API
