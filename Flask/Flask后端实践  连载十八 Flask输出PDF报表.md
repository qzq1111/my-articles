# Flask后端实践  连载十八 Flask输出PDF报表

tips:
- 简单实现Flask输出PDF报表
- 本文基于python3编写
- [代码仓库](https://github.com/qzq1111/flask-resful-example)


## 项目场景
由于项目是工程上的使用，不仅需要对采集的数据进行分析，也需要输出报表，使用程序输出报表极大的简化了报表制作流程。

## Python处理PDF的包
1. reportlab

    ReportLab标记语言（RML）是最强大的代码到PDF工具包，非常简单，是自动化专业发布的最佳选择。 清晰的，类似HTML的语法允许开发人员像网页一样布局数据驱动的文档。[文档地址](https://www.reportlab.com/documentation/)，[用户手册](https://www.reportlab.com/docs/reportlab-userguide.pdf)。
    
2. xhtml2pdf

    xhtml2pdf是一个使用ReportLab Toolkit，HTML5lib和pyPdf的html2pdf转换器。它支持HTML 5和CSS 2.1（以及一些CSS 3）。它完全用纯Python编写，因此它与平台无关。具有Web技能（如HTML和CSS）的用户能够非常快速地生成PDF模板而无需学习新技术。[文档地址](https://xhtml2pdf.readthedocs.io/en/latest/)
3. PyPDF2

    PyPDF2可以提取文档信息（标题，作者，...）、按页拆分文档、逐页合并文档、裁剪页面、合并多个页面到一个页、对pdf文档进行加密解密等等。[文档地址](http://mstamy2.github.io/PyPDF2/)

## ReportLab简单使用

  由于项目上需要绘制PDF，经过比较选择reportlab作为开发。reportlab可以利用模板来生成PDF，也可以手动生成PDF。

1. 安装`pip install reportlab`
2. 简单使用，更多详细内容请看[用户手册](https://www.reportlab.com/docs/reportlab-userguide.pdf)。
    ```python
    from reportlab.lib.colors import HexColor
    from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
    from reportlab.platypus import SimpleDocTemplate, Table, Image, PageBreak, Paragraph
    from reportlab.lib.units import inch
    from reportlab.lib import colors
    from reportlab.pdfbase import pdfmetrics
    from reportlab.pdfbase.ttfonts import TTFont

    # 增加的字体，支持中文显示,需要自行下载支持中文的字体
    pdfmetrics.registerFont(TTFont('SimSun', 'SimSun.ttf'))
    styles = getSampleStyleSheet()
    styles.add(ParagraphStyle(fontName='SimSun', name='SimSun', leading=20, fontSize=12))


    def table_model():
        """
        添加表格
        :return:
        """
        new_img = Image('test.jpg', width=300, height=300)
        base = [
            [new_img, ""],
            ["大类", "小类"],
            ["WebFramework", "django"],
            ["", "flask"],
            ["", "web.py"],
            ["", "tornado"],
            ["Office", "xlsxwriter"],
            ["", "openpyxl"],
            ["", "xlrd"],
            ["", "xlwt"],
            ["", "python-docx"],
            ["", "docxtpl"],
        ]

        style = [
            # 设置字体
            ('FONTNAME', (0, 0), (-1, -1), 'SimSun'),

            # 合并单元格 (列,行)
            ('SPAN', (0, 0), (1, 0)),
            ('SPAN', (0, 2), (0, 5)),
            ('SPAN', (0, 6), (0, 11)),

            # 单元格背景
            ('BACKGROUND', (0, 1), (1, 1), HexColor('#548DD4')),  

            # 字体颜色
            ('TEXTCOLOR', (0, 1), (1, 1), colors.white),  
            # 对齐设置
            ('VALIGN', (0, 0), (-1, -1), 'MIDDLE'),  
            ('ALIGN', (0, 0), (-1, -1), 'CENTER'),  

            # 单元格框线
            ('GRID', (0, 0), (-1, -1), 0.5, colors.grey),
            ('BOX', (0, 0), (-1, -1), 0.5, colors.black),

        ]

        component_table = Table(base, style=style)
        return component_table


    def paragraph_model(msg):
        """
        添加一段文字
        :param msg:
        :return:
        """
        # 设置文字样式
        style = ParagraphStyle(
            name='Normal',
            fontName='SimSun',
            fontSize=50,
        )

        return Paragraph(msg, style=style)


    def image_model():
        """
        添加图片
        :return:
        """
        new_img = Image('test.jpg', width=300, height=300)
        return new_img

    # 更多样式参考用户手册 https://www.reportlab.com/docs/reportlab-userguide.pdf
    data = list()
    # 添加一段文字
    paragraph = paragraph_model("测试添加一段文字")
    data.append(paragraph)
    data.append(PageBreak())  # 分页标识
    # 添加table和图片
    table = table_model()
    data.append(table)
    data.append(PageBreak())  # 分页标识
    img = image_model()
    data.append(img)

    # 设置生成pdf的名字和编剧
    pdf = SimpleDocTemplate("test.pdf", rightMargin=0, leftMargin=0, topMargin=40, bottomMargin=0, )
    # 设置pdf每页的大小
    pdf.pagesize = (9 * inch, 10 * inch)

    pdf.multiBuild(data)

    ```

## Flask结合reportlab使用
- 测试代码

    ```python
    from flask import Flask
    from reportlab.lib.colors import HexColor
    from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
    from reportlab.platypus import SimpleDocTemplate, Table, Image, PageBreak, Paragraph
    from reportlab.lib.units import inch
    from reportlab.lib import colors
    from reportlab.pdfbase import pdfmetrics
    from reportlab.pdfbase.ttfonts import TTFont

    # 增加的字体，支持中文显示,需要自行下载支持中文的字体
    pdfmetrics.registerFont(TTFont('SimSun', 'SimSun.ttf'))
    styles = getSampleStyleSheet()
    styles.add(ParagraphStyle(fontName='SimSun', name='SimSun', leading=20, fontSize=12))


    def table_model():
        """
        添加表格
        :return:
        """
        new_img = Image('test.jpg', width=300, height=300)
        base = [
            [new_img, ""],
            ["大类", "小类"],
            ["WebFramework", "django"],
            ["", "flask"],
            ["", "web.py"],
            ["", "tornado"],
            ["Office", "xlsxwriter"],
            ["", "openpyxl"],
            ["", "xlrd"],
            ["", "xlwt"],
            ["", "python-docx"],
            ["", "docxtpl"],
        ]

        style = [
            # 设置字体
            ('FONTNAME', (0, 0), (-1, -1), 'SimSun'),

            # 合并单元格 (列,行)
            ('SPAN', (0, 0), (1, 0)),
            ('SPAN', (0, 2), (0, 5)),
            ('SPAN', (0, 6), (0, 11)),

            # 单元格背景
            ('BACKGROUND', (0, 1), (1, 1), HexColor('#548DD4')),

            # 字体颜色
            ('TEXTCOLOR', (0, 1), (1, 1), colors.white),
            # 对齐设置
            ('VALIGN', (0, 0), (-1, -1), 'MIDDLE'),
            ('ALIGN', (0, 0), (-1, -1), 'CENTER'),

            # 单元格框线
            ('GRID', (0, 0), (-1, -1), 0.5, colors.grey),
            ('BOX', (0, 0), (-1, -1), 0.5, colors.black),

        ]

        component_table = Table(base, style=style)
        return component_table


    def paragraph_model(msg):
        """
        添加一段文字
        :param msg:
        :return:
        """
        # 设置文字样式
        style = ParagraphStyle(
            name='Normal',
            fontName='SimSun',
            fontSize=50,
        )

        return Paragraph(msg, style=style)


    def image_model():
        """
        添加图片
        :return:
        """
        new_img = Image('test.jpg', width=300, height=300)
        return new_img


    def generate_pdf():
        """
        生成pdf
        :return:
        """
        path = "test.pdf"
        data = list()
        # 添加一段文字
        paragraph = paragraph_model("测试添加一段文字")
        data.append(paragraph)
        data.append(PageBreak())  # 分页标识
        # 添加table和图片
        table = table_model()
        data.append(table)
        data.append(PageBreak())  # 分页标识
        img = image_model()
        data.append(img)

        # 设置生成pdf的名字和编剧
        pdf = SimpleDocTemplate(path, rightMargin=0, leftMargin=0, topMargin=40, bottomMargin=0, )
        # 设置pdf每页的大小
        pdf.pagesize = (9 * inch, 10 * inch)

        pdf.multiBuild(data)
        return path


    app = Flask(__name__)


    @app.route('/testPDF', methods=["GET"])
    def test_pdf():
        """
        测试输出pdf
        :return:
        """
        path = generate_pdf()
        return path

    if __name__ == '__main__':
        app.run()

    ```
- 启动app，访问`http://127.0.0.1:5000/testPDF` 返回生成路径`test.pdf`。然后配合nginx转发即可下载文件

## 总结
- 本篇文章简单介绍了reportlab的使用，具体的使用场景应该与项目需求相结合。