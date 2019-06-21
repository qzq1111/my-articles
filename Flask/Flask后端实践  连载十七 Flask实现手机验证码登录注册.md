# Flask后端实践 连载十七 Flask实现手机验证码登录注册

tips:
- 本文将实现手机验证码登录注册
- 本文基于python3编写
- [代码仓库](https://github.com/qzq1111/flask-resful-example)

## 项目场景

手机号验证码登陆注册，不仅方便了用户操作，也保证用户的数据安全性。

## 实现流程

1. 用户在前端或APP输入手机号点击获取验证码
2. 后端获取到手机号并发送验证码
3. 用户接受到验证码，输入并登陆或注册
4. 后端完成登陆或注册操作，授权用户登陆系统

## 前置条件
1. 安装`flask`、`redis`、`flask-sqlalchemy`、`PyMySQL`、 `aliyun-python-sdk-core`
2. 在阿里云购买[短信服务](https://www.aliyun.com/product/sms)
3. 在短信控制台申请短信模板，成功之后会有短信服务ID、短信服务密钥、短信签名、短信模板等内容。
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
1. `phone_login_or_register.py`中redis相关函数

    ```python
    import redis

    def expire(name, exp=None):
        """
        设置过期时间
        :return:
        """
        expire_in_seconds = exp if exp else 60
        r = redis.StrictRedis('127.0.0.1', '6379', 1)
        r.expire(name, expire_in_seconds)

    def hset(name, key, value):
        """
        设置指定hash表
        :return:
        """
        r = redis.StrictRedis('127.0.0.1', '6379', 1)
        r.hset(name, key, value)

    def hget(name, key):
        """
        读取指定hash表的键值
        """
        r = redis.StrictRedis('127.0.0.1', '6379', 1)
        value = r.hget(name, key)
        return value.decode('utf-8') if value else value
    ```

2. `phone_login_or_register.py`中短信发送相关函数

   ```python
    import uuid
    from aliyunsdkcore.client import AcsClient
    from aliyunsdkcore.request import CommonRequest

    # 阿里云申请成功获取到的内容
    smss = {
        "SMS_ACCESS_KEY_ID": "128548974",  # key ID
        "SMS_ACCESS_KEY_SECRET": "323232",  # 密钥
        "SMS_SIGN_NAME": "设置签名",  # 签名
        "AUTHENTICATION": "SMS_1551323",  # 身份验证模板编码
        "LOGIN_CONFIRMATION": "SMS_155546",  # 登陆确认模板编码
        "LOGIN_EXCEPTION": "SMS_1556546",  # 登陆异常模板编码
        "USER_REGISTRATION": "SMS_1551654625",  # 用户注册模板编码
        "CHANGE_PASSWORD": "SMS_155126456",  # 修改密码模板编码
        "INFORMATION_CHANGE": "SMS_1551265463",  # 信息修改模板编码
    }


    class SendSms(object):

        def __init__(self, phone: str = None, category: str = None, template_param=None):
            """
            :param phone: 发送的手机号
            :param category: 选择短信模板
            :param template_param: 短信验证码或者短信模板中需要替换的变量用字典传入类似：{"code":123456}
            """
            access_key_id = smss.get('SMS_ACCESS_KEY_ID', None)
            access_key_secret = smss.get('SMS_ACCESS_KEY_SECRET', None)
            sign_name = smss.get("SMS_SIGN_NAME", None)

            if access_key_id is None:
                raise ValueError("缺失短信key")

            if access_key_secret is None:
                raise ValueError("缺失短信secret")

            if phone is None:
                raise ValueError("手机号错误")

            if template_param is None:
                raise ValueError("短信模板参数无效")

            if category is None:
                raise ValueError("短信模板编码无效")

            if sign_name is None:
                raise ValueError("短信签名错误")

            self.acs_client = AcsClient(access_key_id, access_key_secret)
            self.phone = phone
            self.category = category
            self.template_param = template_param
            self.template_code = self.template_code()
            self.sign_name = sign_name

        def template_code(self):
            """
            选择模板编码
            :param self.category
            authentication: 身份验证
            login_confirmation: 登陆验证
            login_exception: 登陆异常
            user_registration: 用户注册
            change_password:修改密码
            information_change:信息修改
            :return:
            """
            if self.category == "authentication":
                code = smss.get('AUTHENTICATION', None)
                if code is None:
                    raise ValueError("配置文件中未找到模板编码AUTHENTICATION")
                return code
            elif self.category == "login_confirmation":
                code = smss.get('LOGIN_CONFIRMATION', None)
                if code is None:
                    raise ValueError("配置文件中未找到模板编码LOGIN_CONFIRMATION")
                return code
            elif self.category == "login_exception":
                code = smss.get('LOGIN_EXCEPTION', None)
                if code is None:
                    raise ValueError("配置文件中未找到模板编码LOGIN_EXCEPTION")
                return code
            elif self.category == "user_registration":
                code = smss.get('USER_REGISTRATION', None)
                if code is None:
                    raise ValueError("配置文件中未找到模板编码USER_REGISTRATION")
                return code
            elif self.category == "change_password":
                code = smss.get('CHANGE_PASSWORD', None)
                if code is None:
                    raise ValueError("配置文件中未找到模板编码CHANGE_PASSWORD")
                return code
            elif self.category == "information_change":
                code = smss.get('INFORMATION_CHANGE', None)
                if code is None:
                    raise ValueError("配置文件中未找到模板编码INFORMATION_CHANGE")
                return code
            else:
                raise ValueError("短信模板编码无效")

        def send_sms(self):
            """
            发送短信
            :return:
            """

            sms_request = CommonRequest()

            # 固定设置
            sms_request.set_accept_format('json')
            sms_request.set_domain('dysmsapi.aliyuncs.com')
            sms_request.set_method('POST')
            sms_request.set_protocol_type('https')  # https | http
            sms_request.set_version('2017-05-25')
            sms_request.set_action_name('SendSms')

            # 短信发送的号码列表，必填。
            sms_request.add_query_param('PhoneNumbers', self.phone)
            # 短信签名，必填。
            sms_request.add_query_param('SignName', self.sign_name)

            # 申请的短信模板编码,必填
            sms_request.add_query_param('TemplateCode', self.template_code)

            # 短信模板变量参数 类似{"code":"12345"}，必填。
            sms_request.add_query_param('TemplateParam', self.template_param)

            # 设置业务请求流水号，必填。暂用UUID1代替
            build_id = uuid.uuid1()
            sms_request.add_query_param('OutId', build_id)

            # 调用短信发送接口，返回json
            sms_response = self.acs_client.do_action_with_exception(sms_request)

            return sms_response
   ```

3. `phone_login_or_register.py`中手机验证相关函数

    ```python
    import re

    def check_phone_code(phone: str, code: str) -> bool:
        """
        验证手机号码与验证码是否正确
        :param phone: 手机号码
        :param code: 验证码
        :return:
        """
        re_phone = check_phone(phone)
        if re_phone is None:
            return False
        r_code = hget(re_phone, "code")
        if code == r_code:
            return True
        else:
            return False
    def check_phone(phone: str):
        """
        验证前端传入的字符串是否为手机号码
        :param phone:手机号码
        :return:
        """
        # 判断手机号长度
        if len(str(phone)) == 11:
            # 匹配手机号
            v_phone = re.match(r'^1[3-9][0-9]{9}$', phone)
            if v_phone is None:
                return None
            else:
                phone = v_phone.group()
                return phone
        else:
            return None
    ```
4. `phone_login_or_register.py` 中登陆或注册相关函数
    ```python
    from model import db, UserLoginMethod, User


    def login_or_register(phone):
        """
        登陆或注册
        :param phone:
        :return:
        """

        # 判断用户是否存在与本系统
        user_login = db.session(UserLoginMethod). \
            filter(UserLoginMethod.login_method == "P",
                UserLoginMethod.identification == phone, ).first()

        # 存在则直接返回用户信息
        if user_login:
            user = db.session.query(User.id, User.name).filter(User.id == user_login.user_id).first()
            data = dict(zip(user.keys(), user))
            return data
        # 不存在则先新建用户然后返回用户信息
        else:
            try:
                # 新建用户信息
                new_user = User(name="nickname", age=20)
                db.session.add(new_user)
                db.session.flush()
                # 新建用户登陆方式
                new_user_login = UserLoginMethod(user_id=new_user.id,
                                                login_method="P",
                                                identification=phone,
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

5. `app.py`相关函数
    ```python
    import random
    from datetime import datetime, timedelta
    from flask import Flask, request
    from phone_login_or_register import check_phone, hget,\
     SendSms, hset, expire, login_or_register, check_phone_code

    app = Flask(__name__)


    @app.route('/testGetVerificationCode', methods=["GET"])
    def test_get_verification_code():
        """
        获取手机验证码
        :return:
        """
        now = datetime.now()
        category = request.args.get("category", None) or request.headers.get("category", None)
        phone = request.args.get('account', None)

        # 验证手机号码正确性
        re_phone = check_phone(phone)
        if phone is None or re_phone is None:
            return "手机号码不正确"
        if category is None:
            return "参数缺失"
        try:
            # 获取手机验证码设置时间
            flag = hget(re_phone, 'expire_time')
            if flag is not None:
                flag = datetime.strptime(flag, '%Y-%m-%d %H:%M:%S')
                # 判断是否重复操作
                if (flag - now).total_seconds() < 60:
                    return "请勿频繁操作"
            # 获取随机验证码
            code = "".join([str(random.randint(0, 9)) for _ in range(6)])
            template_param = {"code": code}
            # 发送验证码
            sms = SendSms(phone=re_phone, category=category, template_param=template_param)
            sms.send_sms()
            # 将验证码存入redis，方便接下来的验证
            hset(re_phone, "code", code)
            # 设置重复操作屏障  
            hset(re_phone, "expire_time", (now + timedelta(minutes=1)).strftime('%Y-%m-%d %H:%M:%S'))
             # 设置验证码过去时间
            expire(re_phone, 60 * 3)
            return "成功"
        except Exception as e:
            print(e)
            return "失败"


    @app.route('/testPhoneLoginOrRegister', methods=["POST"])
    def test_phone_login_or_register():
        """
        用户验证码登录或注册
        :return:
        """

        obj = request.get_json(force=True)
        phone = obj.get('account', None)
        code = obj.get('code', None)
        if phone is None or code is None:
            return "参数不正确"
        # 验证手机号和验证码是否正确
        flag = check_phone_code(phone, code)
        if not flag:
            return "验证码错误"

        # 登陆或注册 
        data = login_or_register(phone)

        if data is None:
            return "失败"
        return data


    if __name__ == '__main__':
        app.run()
    ```
6. 测试
   - 访问`http://127.0.0.1:5000/testGetVerificationCode`获取验证码
   - 访问`http://127.0.0.1:5000/testPhoneLoginOrRegister`登陆或注册。

## 总结
1. 本文介绍了手机号验证码登陆注册的流程，具体需求具体分析和实现。
2. 下一篇将介绍Flask输出PDF报表