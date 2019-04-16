# Flask 接口标准化及全球化
 
tips:
- 本文主要解决项目中历史遗留问题，将产品正规化，统一接口返回，实现全球化。
- 本文基于python3编写
-  [代码仓库](https://github.com/mad7802004/flask-resful-example)
 
## 项目场景
某日，本人终于受不了前人挖坑，大怒！！！V1版本项目restful接口返回一团糟，没有固定的格式，前端大佬也是抱怨多次。在本人的积极怂恿下，项目经理决定重写项目，解决诸多历史遗留问题（多得我想哭，自找的）。
 
## 接口统一返回形式
1. 网上收罗一大圈，一般接口响应消息包含错误编码、信息以及数据
2. 类似定义如下：
	```json
	{
		"code":0,
		"msg":"成功",
		"data":null
	}
	```
- code：表示响应状态码
- msg：表示响应消息
- data：表示响应数据
 
## 响应状态码及响应消息
1. 成功类，包含0以及200开头的数字

编码|含义
---|---
0|成功
20001|登录成功
......|....

2. 错误类，400开头的数字

编码|含义
---|---
-1|失败
40000|自定义错误("{model} not have {key} ")
40001|账户或密码错误
40002|参数无效
40003|未找到相关资源
......|....

3. 服务器错误类，500开头的数值

编码|含义
---|---
50001|服务器错误，请联系管理员
50002|计算出错
......|....
 
##  Flask接口返回设计
1. 编写响应码类和响应消息类（code.py）
    ```python
    class ResponseCode(object):
        SUCCESS = 0  # 成功
        FAIL = -1  # 失败
        NO_RESOURCE_FOUND = 40001  # 未找到资源
        INVALID_PARAMETER = 40002  # 参数无效
        ACCOUNT_OR_PASS_WORD_ERR = 40003  # 账户或密码错误
        
    class ResponseMessage(object):
        SUCCESS = "成功"
        FAIL =  "失败"
        NO_RESOURCE_FOUND =  "未找到资源"
        INVALID_PARAMETER =  "参数无效"
        ACCOUNT_OR_PASS_WORD_ERR =  "账户或密码错误"
    ```
	
2. Flask 接口返回实例(app.py)
    ```python
    from flask import Flask,jsonify
    from code import ResponseCode,ResponseMessage

    app =Flask(__name__)

    @app.route("/",methods=["GET"])
    def test():
        test_dict = dict(name="zhang",age=18)
        data=dict(code=ResponseCode.SUCCESS,
                msg=ResponseMessage.SUCCESS,
                data=test_dict)
        return jsonify(data)
    if __name__=="__main__":
        app.run()
    ```
    运行app，访问 http://127.0.0.1:5000/ 返回如下:
    `{"code":0,"data":{"age":18,"name":"zhang"},"msg":"成功"}`

3. 进一步封装响应文本(response.py)，减少重复代码的编写量及保持代码清洁
    ```python
    class ResMsg(object):
        """
        封装响应文本
        """

        def __init__(self, data=None, code=ResponseCode.SUCCESS, 
                    msg=ResponseMessage.SUCCESS):
            self._data = data
            self._msg = msg
            self._code = code

        def update(self, code=None, data=None, msg=None):
            """
            更新默认响应文本
            :param code:响应状态码
            :param data: 响应数据
            :param msg: 响应消息
            :return:
            """
            if code is not None:
                self._code = code
            if data is not None:
                self._data = data
            if msg is not None:
                self._msg = msg

        def add_field(self, name=None, value=None):
            """
            在响应文本中加入新的字段，方便使用
            :param name: 变量名
            :param value: 变量值
            :return:
            """
            if name is not None and value is not None:
                self.__dict__[name] = value

        @property
        def data(self):
            """
            输出响应文本内容
            :return:
            """
            body = self.__dict__
            body["data"] = body.pop("_data")
            body["msg"] = body.pop("_msg")
            body["code"] = body.pop("_code")
            return body
    ```
    app中使用响应文本封装
    ```python
    from flask import Flask, jsonify
    from code import ResponseCode, ResponseMessage
    from response import ResMsg

    app = Flask(__name__)


    @app.route("/", methods=["GET"])
    def test():
        res = ResMsg()
        test_dict = dict(name="zhang", age=18)
        res.update(data=test_dict)
        return jsonify(res.data)


    if __name__ == "__main__":
        app.run()

    ```
    运行app，访问 http://127.0.0.1:5000/ 返回如下:
    `{"code":0,"data":{"age":18,"name":"zhang"},"msg":"成功"}`
4. 实现全球化配置

    实现思路是编写两份不一样的响应消息，前端通过选择语言，后端返回对应语言的消息。因此采用yaml编写响应消息，并加载到Flask应用中，通过语言选择返回不同的语言消息。在我的另外一篇文章[《Flask后端实践  连载一 加载yaml配置文件》](https://blog.csdn.net/qq_22034353/article/details/88591681)中介绍了Flask加载配置yaml文件。
    - 编写msg.yaml，中文响应消息(zh_CN)，英文响应消息（en）
        ```yaml
        zh_CN: &zh
        0: "成功"
        -1: "失败"
        40001: "资源不存在"
        40002: "参数无效"
        40003: "账户或密码错误"
        en:
        <<: *zh
        # 英文编码
        0: "success"
        -1: "fail"
        40001: "No resources found"
        40002: "Invalid argument"
        40003: "Incorrect account or password"

        ```
    - 修改(response.py)，不使用ResponseMessage类
        ```python 
        from code import ResponseCode
        from flask import request, current_app


        class ResMsg(object):
            """
            封装响应文本
            """

            def __init__(self, data=None, code=ResponseCode.SUCCESS, rq=request):
                # 获取请求中语言选择,如果不存在，获取配置中的语言,如果配置中语言不存在,则默认为中文
                self.lang = rq.headers.get("lang", 
                                        current_app.config.get("LANG", "zh_CN") 
                                        )
                self._data = data
                self._msg = current_app.config[self.lang].get(code, None)
                self._code = code

            def update(self, code=None, data=None, msg=None):
                """
                更新默认响应文本
                :param code:响应编码
                :param data: 响应数据
                :param msg: 响应消息
                :return:
                """
                if code is not None:
                    self._code = code
                    # 获取配置中对应语言的响应消息,默认为None
                    self._msg = current_app.config[self.lang].get(code, None)
                if data is not None:
                    self._data = data
                if msg is not None:
                    self._msg = msg

            def add_field(self, name=None, value=None):
                """
                在响应文本中加入新的字段，方便使用
                :param name: 变量名
                :param value: 变量值
                :return:
                """
                if name is not None and value is not None:
                    self.__dict__[name] = value

            @property
            def data(self):
                """
                输出响应文本内容
                :return:
                """
                body = self.__dict__
                body["data"] = body.pop("_data")
                body["msg"] = body.pop("_msg")
                body["code"] = body.pop("_code")
                return body

        ```
    - 修改(app.py)内容，添加读取响应消息配置
        ```python
        from flask import Flask, jsonify
        import yaml
        from code import ResponseCode
        from response import ResMsg

        app = Flask(__name__)

        with open("msg.yaml", encoding="utf-8") as f:
            msg_conf = yaml.safe_load(f)
        app.config.update(msg_conf)

        @app.route("/", methods=["GET"])
        def test():
            res = ResMsg()
            test_dict = dict(name="zhang", age=18)
            # 此处只需要填入响应状态码,即可获取到对应的响应消息
            res.update(code=ResponseCode.SUCCESS, data=test_dict)
            return jsonify(res.data)


        if __name__ == "__main__":
            app.run()

        ```
    运行app，访问 http://127.0.0.1:5000/  ，利用postman设置不同的请求头，返回如下：
    中文(headers中**lang=zh_CN**):
    `{ "code": 0,"data": {"age": 18,"name": "zhang"},"lang": "zh_CN","msg": "成功"}`
    英文(headers中**lang=en**):
    `{ "code": 0,"data": {"age": 18,"name": "zhang"},"lang": "en","msg": "success"}`

    **注意**：
    1. headers中的lang参数值需要和msg.yaml中的语言设置一样
    2. msg.yaml中的响应状态码应和ResponseCode编码一致
    
## 总结
- 通过定制统一返回接口，解决了项目中接口响应文本混乱的情况，提高前后端的开发效率
- 实现了响应消息可配置化，极大的方便后续开发其他语言版本的接口
- 实现了响应文本统一封装，减少重复代码，提高代码简洁度
- 下一篇文章将介绍如何实现FlaskRestful响应接口封装及自定义json返回类型