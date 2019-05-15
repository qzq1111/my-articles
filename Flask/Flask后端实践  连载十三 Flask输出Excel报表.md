# Flask后端实践  连载十三 Flask输出Excel报表
tips:
- 简单实现Flask输出Excel报表
- 本文基于python3编写
- [代码仓库](https://github.com/qzq1111/flask-resful-example)

## 项目场景
由于项目是工程上的使用，不仅需要对采集的数据进行分析，也需要输出报表，使用程序输出报表极大的简化了报表制作流程。
## Python处理Excel的包
1. openpyxl

    用于读取和写入Excel 2010文件的推荐包（即：.xlsx）。[文档地址](https://openpyxl.readthedocs.io/en/stable/)
2. xlsxwriter

    用于编写数据，格式化信息，特别是Excel 2010格式的图表的替代软件包（即：.xlsx）。[文档地址](https://xlsxwriter.readthedocs.io/)
3. xlrd

    此包用于从旧的Excel文件中读取数据和格式信息（即：.xls）。[文档地址](https://xlrd.readthedocs.io/en/latest/)
4. xlwt

    此包用于将数据和格式信息写入旧的Excel文件（即：.xls）。[文档地址](http://xlwt.readthedocs.io/en/latest/)
5. xlutils

    该软件包收集需要xlrd和xlwt的实用程序，包括复制，修改或过滤现有excel文件的功能。一般来说，openpyxl现在涵盖了这些用例！[文档地址](https://xlutils.readthedocs.io/en/latest/)
## xlsxwriter简单使用

项目上所需的报表，图表比较多，且比较复杂。所有就采用了绘图方面更加完善的`xlsxwriter`包

1. 安装`pip install xlsxwriter`
2. 简单使用，更多详细内容请看[官方文档](https://xlsxwriter.readthedocs.io/)。
    ```python
    import xlsxwriter

    # 新建excel文本
    workbook = xlsxwriter.Workbook("test.xlsx")
    # 添加一个sheet
    worksheet = workbook.add_worksheet("test1")
    # 设置列宽
    worksheet.set_column('A:A', 20)

    # 添加字体加粗样式
    bold = workbook.add_format({'bold': True})

    # 写入数据
    worksheet.write('A1', 'Hello')

    # 写入数据并使用样子
    worksheet.write('A2', 'World', bold)

    # 使用数字标识单元格位置（行，列）
    worksheet.write(2, 0, 123)
    worksheet.write(3, 0, 123.456)

    # 按行依次写入数据
    worksheet.write_column(4, 0, [1, 2, 3, 4])

    # 按列依次写入数据
    worksheet.write_row(1, 1, [1, 2, 3, 4])

    # 添加图表
    chart = workbook.add_chart({'type': 'column'})

    # 图表数据来源
    chart.add_series({'values': ["test1",  # worksheet的名字。即sheet_name
                                4, 0, 7, 0  # 数据位置
                                ]})
    # 插入的表格位置
    worksheet.insert_chart('B3', chart)

    # 关闭excel文本并输出到指定位置。如果不调用改方法，无法输出excel
    workbook.close()
    ```
## Flask结合xlsxwriter使用
- 测试代码
    ```python
    import xlsxwriter
    from flask import Flask


    def write():
        path = "test.xlsx"
        # 新建excel文本
        workbook = xlsxwriter.Workbook(path)
        # 添加一个sheet
        worksheet = workbook.add_worksheet("test1")
        # 设置列宽
        worksheet.set_column('A:A', 20)

        # 添加字体加粗样式
        bold = workbook.add_format({'bold': True})

        # 写入数据
        worksheet.write('A1', 'Hello')

        # 写入数据并使用样子
        worksheet.write('A2', 'World', bold)

        # 使用数字标识单元格位置（行，列）
        worksheet.write(2, 0, 123)
        worksheet.write(3, 0, 123.456)

        # 按行依次写入数据
        worksheet.write_column(4, 0, [1, 2, 3, 4])

        # 按列依次写入数据
        worksheet.write_row(1, 1, [1, 2, 3, 4])

        # 添加图表
        chart = workbook.add_chart({'type': 'column'})

        # 图表数据来源
        chart.add_series({'values': ["test1",  # worksheet的名字。即sheet_name
                                    4, 0, 7, 0  # 数据位置
                                    ]})
        # 插入的表格位置
        worksheet.insert_chart('B3', chart)

        # 关闭excel文本并输出到指定位置。如果不调用改方法，无法输出excel
        workbook.close()
        return path


    app = Flask(__name__)


    @app.route('/testExcel', methods=["GET"])
    def test_excel():
        """
        测试输出excel
        :return:
        """
        path = write()
        return path


    if __name__ == '__main__':
        app.run()

    ```

 - 启动app，访问`http://127.0.0.1:5000/testExcel` 返回生成路径`test.xlsx`。然后配合nginx转发即可下载文件

## 总结
- 本篇文章简单介绍了xlsxwriter的使用，具体的使用场景应该与项目需求相结合。
- 下一篇将介绍Flask输出Word报表