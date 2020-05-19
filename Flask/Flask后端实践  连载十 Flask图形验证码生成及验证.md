# Flask后端实践  连载十 Flask图形验证码生成及验证
tips:
 - 本文使用Pillow生成图形验证码，Flask图形验证码验证
 - 本文基于python3编写
 - [代码仓库](https://github.com/qzq1111/flask-resful-example)

## Pillow简单使用
1. 安装`pip install  pillow`
2. Image类，打开图像或新建空白图像
    ```python
    from PIL import Image
    # 第一种：打开已有图片
    im = Image.open("test.png")  # 打开图片
    print(im.format, im.size, im.mode)  # 图片格式、图片大小、图片模式
    im.show()  # 显示图片
    # 第二种：新建图片
    im = Image.new('RGB', (200, 200), 'white')  # 图片模式、图片大小、图片颜色
    im.thumbnail((100, 100))  # 图片缩放
    im.save("test2.png")  # 保存图片
    im.show()  # 显示图片
    ```
3. ImageFont类，设置字体
    ```python
    from PIL import Image, ImageFont, ImageDraw
    im = Image.new('RGB', (200, 50), 'white')  # 图片模式、图片大小、图片颜色
    # font = ImageFont.load_default()  # 加载默认字体，该方法无法指定字体大小，只能输出英文字母和数值
    # 加载指定的字体,并指定字体大小，支持中文，需要将字体放入当前目录。
    font = ImageFont.truetype("SIMYOU.TTF", size=20)  
    draw = ImageDraw.Draw(im)  # 将图片转换为可编辑
    draw.text((50, 10), "测试", font=font, fill="red")  # 输入文字
    im.save("test2.png")  # 保存图片
    im.show()  # 显示图片
    ```

4. ImageDraw类，自定义画图
    ```python
    from PIL import Image, ImageFont, ImageDraw
    im = Image.new('RGB', (400, 200), 'white')  # 图片模式、图片大小、图片颜色
    # font = ImageFont.load_default()  # 加载默认字体，该方法无法指定字体大小，只能输出英文字母和数值
    # 加载指定的字体,并指定字体大小，支持中文，需要将字体放入当前目录。
    font = ImageFont.truetype("SIMYOU.TTF", size=20)  
    draw = ImageDraw.Draw(im)  # 将图片转换为可编辑
    draw.text((50, 10), "测试", font=font, fill="red")  # 输入文字
    draw.line((0, 0) + im.size, fill=120)  # 画线
    # 其他画图可以参考官网
    im.save("test2.png")  # 保存图片
    im.show()  # 显示图片
    ```

## Pillow生成图形验证码
```python
import base64
import io
import random
import string

from PIL import Image, ImageFont, ImageDraw

# 新建一个图层
im = Image.new('RGB', (50, 12), 'white')
# 加载默认字体
font = ImageFont.load_default()
# 获取draw对象
draw = ImageDraw.Draw(im)
# 设置随机4位数字验证码
code = ''.join(random.sample(string.digits, 4))
# 随机颜色
random_color = lambda: (random.randint(32, 127),
                        random.randint(32, 127),
                        random.randint(32, 127))
print(code)
# 将数字输出到图片
for item in range(4):
    draw.text(
        (6 + random.randint(-3, 3) + 10 * item,
         2 + random.randint(-2, 2)),
        text=code[item], fill=random_color(), font=font)

# 保存图片
im.save("test3.png", format="JPEG")

# 重新设置图片大小
im.resize((100, 24))
# 图片转换为Base64字符串
buffered = io.BytesIO()
im.save(buffered, format="JPEG")
img_str = b"data:image/png;base64," + base64.b64encode(buffered.getvalue())
print(img_str)
```
## Flask图形验证码验证
1. 封装图片生成类`util.py` 
    ```python
    import base64
    import io
    import random
    import string
    from PIL import Image, ImageFont, ImageDraw
    
    class CaptchaTool(object):
        """
        生成图片验证码
        """

        def __init__(self, width=50, height=12):

            self.width = width
            self.height = height
            # 新图片对象
            self.im = Image.new('RGB', (width, height), 'white')
            # 字体
            self.font = ImageFont.load_default()
            # draw对象
            self.draw = ImageDraw.Draw(self.im)

        def draw_lines(self, num=3):
            """
            划线
            """
            for num in range(num):
                x1 = random.randint(0, self.width / 2)
                y1 = random.randint(0, self.height / 2)
                x2 = random.randint(0, self.width)
                y2 = random.randint(self.height / 2, self.height)
                self.draw.line(((x1, y1), (x2, y2)), fill='black', width=1)

        def get_verify_code(self):
            """
            生成验证码图形
            """
            # 设置随机4位数字验证码
            code = ''.join(random.sample(string.digits, 4))
            # 绘制字符串
            for item in range(4):
                self.draw.text((6 + random.randint(-3, 3) + 10 * item, 2 + random.randint(-2, 2)),
                            text=code[item],
                            fill=(random.randint(32, 127),
                                    random.randint(32, 127),
                                    random.randint(32, 127))
                            , font=self.font)
            # 划线
            # self.draw_lines()
            # 重新设置图片大小
            self.im = self.im.resize((100, 24))  
            # 图片转为base64字符串
            buffered = io.BytesIO()
            self.im.save(buffered, format="JPEG")
            img_str = b"data:image/png;base64," + base64.b64encode(buffered.getvalue())
            return img_str, code
    ```
2. 新建Flask app `app.py`
    ```python
    from flask import Flask, session, request
    from util import CaptchaTool

    app = Flask(__name__)
    app.config["SECRET_KEY"] = "156456dsadasd"

    @app.route('/testGetCaptcha', methods=["GET"])
    def test_get_captcha():
        """
        获取图形验证码
        :return:
        """
        new_captcha = CaptchaTool()
        # 获取图形验证码
        img, code = new_captcha.get_verify_code()
        # 存入session
        session["code"] = code
        return img

    @app.route('/testVerifyCaptcha', methods=["POST"])
    def test_verify_captcha():
        """
        验证图形验证码
        :return:
        """
        obj = request.get_json(force=True)
        # 获取用户输入的验证码
        code = obj.get('code', None)
        # 获取session中的验证码
        s_code = session.get("code", None)
        print(code, s_code)
        if not all([code, s_code]):
            return "参数错误"
        if code != s_code:
            return "验证码错误"
        return "验证成功"

    if __name__ == "__main__":
        app.run()
    ```

3. 测试
   - 通过postman访问`http://127.0.0.1:5000/testGetCaptcha`获取验证码
   - 将获取的字符串直接放入浏览器查看图片，再获取到图片的同时，也会在返回头中添加session
   - postman发送POST json到`http://127.0.0.1:5000/testVerifyCaptcha` {"code":"图片数字"}，其中将请求头设置刚获取到session。如果是浏览器，会自动设置。

## 总结
- 简单的介绍了`pillow`的使用,并生成了数字验证码，也可以生成不同类型的验证码。
- 通过画线、高斯模糊等手段可以加强验证码难度。
  