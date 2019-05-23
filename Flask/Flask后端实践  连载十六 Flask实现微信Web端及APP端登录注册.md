# Flask后端实践  连载十六 Flask实现微信Web端及APP端登录注册

tips:
- 本文将实现微信Web端和APP端登陆注册
- 本文基于python3编写
- [代码仓库](https://github.com/qzq1111/flask-resful-example)

## 项目场景

某天，项目经理说，项目上除了本身自带的登陆注册，也需要第三方的登陆注册。方便用户使用我们的产品。于是便开始加上微信登陆注册的功能。

## 实现流程

1. 前端页面拉取微信授权二维码
2. 用户扫码确认授权。
3. 前端将授权码发送到后端。
4. 后端将授权码发送到微信平台验证，获取用户信息及授权信息。
5. 比较本平台用户信息，实现用户登陆和注册。

## 前置条件
1. 安装`flask`、`flask-sqlalchemy`、`PyMySQL`
2. 在[微信开放平台](https://open.weixin.qq.com/)申请了账户并成为开发者。
3. 创建网站应用和移动应用，申请成功之后会下发`appid`和`secret`，web和app的`appid`、`secret`不一样。
4. 数据库设计
    - user（用户表，存储用户信息）
    
        |字段|类型|含义|
        |---|---|---|
        |id|int|用户主键ID|
        |name|string|用户名称|
        |age|int|年龄|
        |...|...|...|
        
    - user_login_method（用户登陆方式表，存储不同登陆方式）
    
      |字段|类型|含义|
      |---|---|---|
      |id|int|用户登陆方式主键ID|
      |user_id|int|用户主键ID|
      |login_method|string|用户登陆方式，WX微信，P手机|
      |identification|string|用户登陆标识，微信ID或手机号|
      |access_code|string|用户登陆通行码，密码或token|
      |...|...|...|
      
5. 映射数据库映射数据库到`model.py`
    ```python
    from flask_sqlalchemy import SQLAlchemy

    db = SQLAlchemy()

    class User(db.Model):
        """
        用户表
        """
        __tablename__ = 'user'
        id = db.Column(db.Integer, autoincrement=True, primary_key=True)
        # 用户姓名
        name = db.Column(db.String(20), nullable=False) 
        # 用户年龄
        age = db.Column(db.Integer, nullable=False)  


    class UserLoginMethod(db.Model):
        """
        用户登陆验证表
        """
        __tablename__ = 'user_login_method'
        # 用户登陆方式主键ID
        id = db.Column(db.Integer, autoincrement=True, primary_key=True)  
        # 用户主键ID
        user_id = db.Column(db.Integer, nullable=False) 
        # 用户登陆方式，WX微信，P手机 
        login_method = db.Column(db.String(36), nullable=False) 
        # 用户登陆标识，微信ID或手机号
        identification = db.Column(db.String(36), nullable=False)  
        # 用户登陆通行码，密码或token
        access_code = db.Column(db.String(36), nullable=True) 
    ```

## 实现过程
1. 前端拉取微信认证授权二维码，[参考网站应用微信登录开发指南](https://open.weixin.qq.com/cgi-bin/showdocument?action=dir_list&t=resource/res_list&verify=1&id=open1419316505&token=&lang=zh_CN)。此处只需要第一步，用户扫码确认之后获取到code传入后端。
2. 获取授权凭证`wx_login_or_register.get_access_code`
    
    ```python
    import json
    from urllib import parse, request

    from model import UserLoginMethod, User, db

    def get_access_code(code, flag):
        """
        获取微信授权码
        :param code:前端或app拉取的到临时授权码
        :param flag:web端或app端
        :return:None 或 微信授权数据
        """
        # 判断是web端登陆还是app端登陆，采用不同的密钥。
        if flag == "web":
            appid = "web_appid"
            secret = "web_secret"
        elif flag == "app":
            appid = "app_appid"
            secret = "app_secret"
        else:
            return None
        try:
            # 把查询条件转成url中形式
            fields = parse.urlencode(
                {"appid": appid, "secret": secret,
                "code": code, "grant_type": "authorization_code"}
            )
            # 拼接请求链接
            url = 'https://api.weixin.qq.com/sns/oauth2/access_token?{}'.format(fields) 
            print(url)
            req = request.Request(url=url, method="GET")
            # 请求数据
            res = request.urlopen(req, timeout=10)
            # 解析数据
            access_data = json.loads(res.read().decode())
            print(access_data)
        except Exception as e:
            print(e)
            return None

        # 拉取微信授权成功返回
        # {
        # "access_token": "ACCESS_TOKEN", "expires_in": 7200,"refresh_token": "REFRESH_TOKEN",
        # "openid": "OPENID","scope": "SCOPE"
        # }

        if "openid" in access_data:
            return access_data

        # 拉取微信授权失败
        # {
        # "errcode":40029,"errmsg":"invalid code"
        # }
        else:
            return None

    ```
3. 获取微信用户信息`wx_login_or_register.get_wx_user_info`

    ```python
    def get_wx_user_info(access_data: dict):
        """
        获取微信用户信息
        :return:
        """
        openid = access_data.get("openid")
        access_token = access_data.get("access_token")
        try:
            # 把查询条件转成url中形式
            fields = parse.urlencode({"access_token": access_token, "openid": openid})
            # 拼接请求链接
            url = 'https://api.weixin.qq.com/sns/userinfo?{}'.format(fields)
            print(url)
            req = request.Request(url=url, method="GET")
            # 请求数据,超时10s
            res = request.urlopen(req, timeout=10)
            # 解析数据
            wx_user_info = json.loads(res.read().decode())
            print(wx_user_info)
        except Exception as e:
            print(e)
            return None

        # 获取成功
        # {
        # "openid":"OPENID",
        # "nickname":"NICKNAME",
        # "sex":1,
        # "province":"PROVINCE",
        # "city":"CITY",
        # "country":"COUNTRY",
        # "headimgurl": "test.png",
        # "privilege":[
        # "PRIVILEGE1",
        # "PRIVILEGE2"
        # ],
        # "unionid": " o6_bmasdasdsad6_2sgVt7hMZOPfL"
        #
        # }
        if "openid" in wx_user_info:
            return wx_user_info
        #  获取失败
        # {"errcode":40003,"errmsg":"invalid openid"}
        else:
            return None
    ```
4. 验证本地用户信息`wx_login_or_register.login_or_register`，关键的是比较微信用户信息中的`unionid`。
    ```python
    def login_or_register(wx_user_info):
        """
        验证该用户是否注册本平台，如果未注册便注册后登陆，否则直接登陆。
        :param wx_user_info:拉取到的微信用户信息
        :return:
        """
        # 微信统一ID
        unionid = wx_user_info.get("unionid")
        # 用户昵称
        nickname = wx_user_info.get("nickname")
        # 拉取微信用户信息失败
        if unionid is None:
            return None

        # 判断用户是否存在与本系统
        user_login = db.session(UserLoginMethod). \
            filter(UserLoginMethod.login_method == "WX",
                UserLoginMethod.identification == unionid, ).first()
        # 存在则直接返回用户信息
        if user_login:
            user = db.session.query(User.id, User.name).\
                filter(User.id == user_login.user_id).first()
            data = dict(zip(user.keys(), user))
            return data
        # 不存在则先新建用户然后返回用户信息
        else:
            try:
                # 新建用户信息
                new_user = User(name=nickname, age=20)
                db.session.add(new_user)
                db.session.flush()
                # 新建用户登陆方式
                new_user_login = UserLoginMethod(user_id=new_user.id,
                                                login_method="WX",
                                                identification=unionid,
                                                access_code=None)
                db.session.add(new_user_login)
                db.session.flush()
                # 提交
                db.session.commit()
            except Exception as e:
                print(e)
                return None
            
            data = dict(id=new_user.id, name=User.name)
            return data
    ```


5. Flask接口
    ```python

    from flask import Flask
    from wx_login_or_register import get_access_code, get_wx_user_info, login_or_register
    from model import db

    app = Flask(__name__)
    app.config["SQLALCHEMY_DATABASE_URI"]='mysql+pymysql://root:mad123@localhost:3306/test?charset=utf8mb4'
    # 注册数据库连接
    db.app = app
    db.init_app(app)
    
    @app.route("/testWXLoginOrRegister",methods=["GET"])
    def test_wx_login_or_register():
        """
        测试微信登陆注册
        :return:
        """
        # 前端获取到的临时授权码
        code = request.args.get("code") 
        # 标识web端还是app端登陆或注册
        flag = request.args.get("flag")

        # 参数错误
        if code is None or flag is None:
            return "参数错误"

        # 获取微信用户授权码
        access_code = get_access_code(code=code, flag=flag)
        if access_code is None:
            return "获取微信授权失败"

        # 获取微信用户信息
        wx_user_info = get_wx_user_info(access_data=access_code)
        if wx_user_info is None:
            return "获取微信授权失败"

        # 验证微信用户信息本平台是否有，
        data = login_or_register(wx_user_info=wx_user_info)
        if data is None:
            return "登陆失败"
        return data

    if  ___name__ =="__main__":
        app.run()

    ```
6. 测试访问`http://127.0.0.1:5000/testWXLoginOrRegister`即可。
## 总结
- 本文大致实现了微信web和app端的登陆注册，具体需求具体分析。
- 下一篇将介绍Flask实现手机验证码登录注册
