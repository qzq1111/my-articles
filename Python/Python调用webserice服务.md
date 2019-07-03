# Python 调用webservice服务

tips:
- 本文代码基于python3编写
- [代码仓库](https://github.com/qzq1111/python-summary)


## 前置条件
1. Python访问webservice接口用到的工具包是`suds`，但是由于该工具包没有在维护了，本文使用`suds`的分支项目`suds-community`
2. 安装`suds-community`，`pip install suds-community`
3. 免费的webservice网站，`http://www.webxml.com.cn/zh_cn/web_services.aspx`。

## 使用suds-community调用webservice服务

```python
from suds.client import Client

# 连接到webservice服务，获取查询手机号码归属地服务方法
client = Client('http://ws.webxml.com.cn/WebServices/MobileCodeWS.asmx?wsdl')
# 输出服务方法
print(client)
# Suds ( https://fedorahosted.org/suds/ )  version: 0.8.3
#
# Service ( MobileCodeWS ) tns="http://WebXml.com.cn/"
#    Prefixes (1)
#       ns0 = "http://WebXml.com.cn/"
#    Ports (2):
#       (MobileCodeWSSoap)
#          Methods (2):
#             getDatabaseInfo()
#             getMobileCodeInfo(xs:string mobileCode, xs:string userID)
#          Types (1):
#             ArrayOfString
#       (MobileCodeWSSoap12)
#          Methods (2):
#             getDatabaseInfo()
#             getMobileCodeInfo(xs:string mobileCode, xs:string userID)
#          Types (1):
#             ArrayOfString

# 一共有两个方法 getDatabaseInfo()   getMobileCodeInfo(xs:string mobileCode, xs:string userID)
# 具体含义请看 http://ws.webxml.com.cn/WebServices/MobileCodeWS.asmx

# 查询手机号码归属地
print(client.service.getMobileCodeInfo("18300000000",""))
# 18300000000：广东 深圳 广东移动全球通卡
```

## 踩坑点及解决方法
1. 下面的代码触发`suds.TypeNotFound: Type not found: '(schema, http://www.w3.org/2001/XMLSchema, )`错误，错误的原因是没有正确的引入命名空间。解决办法，用浏览器打开webservice服务链接，找到webservice服务中的`targetNamespace`，将它的只添加到过滤的命名空间就能解决问题。但是一旦使用这个方法。速度会变得很慢（原因未知）。
    ```python
    # 引发错误
    from suds.client import Client
    # 连接到webservice服务，获取查询天气服务方法
    client = Client('http://ws.webxml.com.cn/WebServices/WeatherWS.asmx?wsdl')
    print(client)
    ```

    ```python
    # 解决错误
    from suds.client import Client
    from suds.xsd.doctor import ImportDoctor, Import

    # 导入正确的命名空间。
    imp = Import('http://www.w3.org/2001/XMLSchema', location='http://www.w3.org/2001/XMLSchema.xsd')
    imp.filter.add('http://WebXml.com.cn/')
    doctor = ImportDoctor(imp)
    client = Client('http://ws.webxml.com.cn/WebServices/WeatherWS.asmx?wsdl', doctor=doctor)
    print(client)
    ```
2. 关于参数传递，如果webservice服务需要传递多个参数，按照顺序填入到调用服务中，最好的解决方法是使用字典传入。如下所示：
    ```python
    from suds.client import Client
    from suds.xsd.doctor import ImportDoctor, Import

    # 导入正确的命名空间。
    imp = Import('http://www.w3.org/2001/XMLSchema', location='http://www.w3.org/2001/XMLSchema.xsd')
    imp.filter.add('http://WebXml.com.cn/')
    doctor = ImportDoctor(imp)
    client = Client('http://ws.webxml.com.cn/WebServices/WeatherWS.asmx?wsdl', doctor=doctor)
    # 使用字典传入参数
    print(client.service.getSupportCityString(**{"theRegionCode": "四川"}))
    ```
   
## 总结
- 本文使用了`suds-community`来调用webservice服务
- 解决了`Type not found: '(schema, http://www.w3.org/2001/XMLSchema, )`问题
- 参数转递使用字典