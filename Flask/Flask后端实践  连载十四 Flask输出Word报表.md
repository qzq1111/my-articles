# Flask后端实践  连载十四 Flask输出Word报表

tips:
- 简单实现Flask输出Word报表
- 本文基于python3编写
- [代码仓库](https://github.com/qzq1111/flask-resful-example)

## 项目场景
由于项目是工程上的使用，不仅需要对采集的数据进行分析，也需要输出报表，使用程序输出报表极大的简化了报表制作流程。

## Python处理Word的包
1. python-docx

    python-docx是一个用于创建和更新Microsoft Word（.docx）文件的Python库。[文档地址](https://python-docx.readthedocs.io/en/latest/)

2. docxtpl

    主要包含两个包，分别是python-docx用于读取，编写和创建子文档。jinja2用于管理插入模板docx的标签。通过预先设定的模板文件，生成需要的文件。[文档地址](https://docxtpl.readthedocs.io/en/latest/)

## docxtpl的简单使用

项目上的Word报表定制化比较高。通常由用户提供模板，我们这边只需要在对应位置填写数据。因此项目上采用`docxtpl`来定制化Word报表

1. 安装`pip install docxtpl`
2. 简单使用，更多详细内容请看[官方文档](https://docxtpl.readthedocs.io/en/latest/)。

   - 新建测试模板`test.docx`，设置模板样子，并填入相关参数。
   - 编写渲染测试代码`test.py`
    
        ```python
        from docxtpl import DocxTemplate, InlineImage
        from docx.shared import Mm

        # 读取指定位置的模板文件
        doc = DocxTemplate("test.docx")
        # 渲染的内容
        context = {
            # 标题
            'title': "人员信息",
            # 表格
            'table': [
                {"name": "小李", "age": 11},
                {"name": "小张", "age": 21},
                {"name": "小张", "age": 20},
                {"name": "小张1", "age": 10},
                {"name": "小张2", "age": 30},
                {"name": "小张3", "age": 40},
            ],
            # 页眉
            'header': 'xxx公司人员信息管理',
            # 页脚
            'footer': '1',
            # 图片
            'image': InlineImage(doc, 'test.jpg', height=Mm(10)),
        }
        # 渲染模板
        doc.render(context)
        # 保存渲染的文件
        doc.save("generated_doc.docx")
        ```

3. 关于表格动态合并、表格设置可以参考[这里](https://blog.csdn.net/weixin_42670653/article/details/81531668)


## Flask结合docxtpl使用
- 测试代码
    ```python
    from docxtpl import DocxTemplate, InlineImage
    from docx.shared import Mm
    from flask import Flask


    def write():
        path = "generated_doc.docx"
        # 读取指定位置的模板文件
        doc = DocxTemplate("test.docx")
        # 渲染的内容
        context = {
            # 标题
            'title': "人员信息",
            # 表格
            'table': [
                {"name": "小李", "age": 11},
                {"name": "小张", "age": 21},
                {"name": "小张", "age": 20},
                {"name": "小张1", "age": 10},
                {"name": "小张2", "age": 30},
                {"name": "小张3", "age": 40},
            ],
            # 页眉
            'header': 'xxx公司人员信息管理',
            # 页脚
            'footer': '1',
            # 图片
            'image': InlineImage(doc, 'test.jpg', height=Mm(10)),
        }
        # 渲染模板
        doc.render(context)
        # 保存渲染的文件
        doc.save(path)
        return path


    app = Flask(__name__)


    @app.route('/testWord', methods=["GET"])
    def test_word():
        """
        测试输出word
        :return:
        """
        path = write()
        return path


    if __name__ == '__main__':
        app.run()


    ```

 - 启动app，访问`http://127.0.0.1:5000/testWord` 返回生成路径`generated_doc.docx`。然后配合nginx转发即可下载文件

## 总结
- 本篇文章简单介绍了docxtpl的使用，具体的使用场景应该与项目需求相结合。
- 下一篇将解决自关联无限层级(目录树)处理