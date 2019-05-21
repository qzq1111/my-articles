# Flask后端实践  连载十五 实现自关联无限层级生成目录树

tips:
- 本文主要解决自关联无限层级生成目录树。
- 本文基于python3编写
- [代码仓库](https://github.com/qzq1111/flask-resful-example)


## 问题重现
项目上有组织结构的管理，所有需要将其输出成一颗目录树。
- 数据库数据组织方式，采用自关联的方式，示例如下：
  
    |id|father_id|name|
    |---|---|--|
    |1|null|01|
    |2|1|0101|
    |3|1|0102|
    |4|1|0103|
    |...|...|...|
    
- 目录树，示例如下：
  ```
  json:

  {"name":"01","id":1,"child":[
      {"name":"0101","id":2,"child":[...]},
      {"name":"0102","id":3,"child":[...]},
      {"name":"0103","id":4,"child":[...]},
      ...
      ]
  }

  目录树：
   -01
     |__0101
     |    |__...
     |
     |__0102
     |     |__...
     |
     |__0103
     |     |__...
     |
     |__...

  ```

## 问题解决
1. 实现思路：
   - 寻找到根节点，放入到列表A中。
   - 寻找共同的父节点，并放入到字典B。
   - 循环遍历根节点A列表，在共同父节点字典B中取出对应的子节点列表C。
   - 依次递归遍历子节点列表C，直到没有子节点列表为止。

2. 实现代码:
   - 寻找根节点`find_root_node`
       ```python
       def find_root_node(data: list) -> list:
           """
           查找根节点
           :param data:原始数据
           :return:根节点
           """
           # root_node = filter(lambda x: x["father_id"] is None, data)

           root_node = list()
           for node in data:
               # 假定father_id是None就是根节点
               # 例如有些数据库设计会直接把根节点标识出来。
               if node["father_id"] is None:
                   root_node.append(node)
           return root_node
       ```
   - 寻找共同的父节点`find_common_node`
       ```python
       def find_common_node(data: list) -> dict:
           """
           寻找共同的父节点
           :param data: 原始数据
           :return: 共同的父节点字典
           """
           common_node = dict()

           for node in data:
               father_id = node.get("father_id")
               # 排除根节点情况
               if father_id is not None:
                   # 如果父节点ID不在字典中则添加到字典中
                   if father_id not in common_node:
                       common_node[father_id] = list()
                   common_node[father_id].append(node)
           return common_node
       ```
   - 生成目录树json`build_tree`和`find_child`
       ```python
       def build_tree(root_node: list, common_node: dict) -> list:
           """
           生成目录树
           :param root_node: 根节点列表
           :param common_node: 共同父节点字典
           :return:
           """
           tree = list()
           for root in root_node:
               # 生成字典
               base = dict(name=root["name"], id=root["id"], child=list())
               # 遍历查询子节点
               find_child(base["id"], base["child"], common_node)
               # 添加到列表
               tree.append(base)
           return tree


       def find_child(father_id: int, child_node: list, common_node: dict):
           """
           查找子节点
           :param father_id:父级ID
           :param child_node: 父级孩子节点
           :param common_node: 共同父节点字典
           :return:
           """
           # 获取共同父节点字典的子节点数据
           child_list = common_node.get(father_id, [])
           for item in child_list:
               # 生成字典
               base = dict(name=item["name"], id=item["id"], child=list())
               # 遍历查询子节点
               find_child(item["id"], base["child"], common_node)
               # 添加到列表
               child_node.append(base)
       ```
   - 测试
     ```python
       data = [
       {"id": 1, "father_id": None, "name": "01"},
       {"id": 2, "father_id": 1, "name": "0101"},
       {"id": 3, "father_id": 1, "name": "0102"},
       {"id": 4, "father_id": 1, "name": "0103"},
       {"id": 5, "father_id": 2, "name": "010101"},
       {"id": 6, "father_id": 2, "name": "010102"},
       {"id": 7, "father_id": 2, "name": "010103"},
       {"id": 8, "father_id": 3, "name": "010201"},
       {"id": 9, "father_id": 4, "name": "010301"},
       {"id": 10, "father_id": 9, "name": "01030101"},
       {"id": 11, "father_id": 9, "name": "01030102"},
       ]

       root_node = find_root_node(data)
       common_node = find_common_node(data)

       tree = build_tree(root_node, common_node)
       print(tree)
       # 输出
       # [{'name': '01', 'id': 1, 'child': [{'name': '0101', 'id': 2, 'child': [{'name': '010101', 'id': 5, 'child': []}, {'name': '010102', 'id': 6, 'child': []}, {'name': '010103', 'id': 7, 'child': []}]}, {'name': '0102', 'id': 3, 'child': [{'name': '010201', 'id': 8, 'child': []}]}, {'name': '0103', 'id': 4, 'child': [{'name': '010301', 'id': 9, 'child': [{'name': '01030101', 'id': 10, 'child': []}, {'name': '01030102', 'id': 11, 'child': []}]}]}]}]
     ```
## 封装为类
- 代码
    ```python
    class Tree(object):
        def __init__(self, data):
            self.data = data
            self.root_node = list()
            self.common_node = dict()
            self.tree = list()

        def find_root_node(self) -> list:
            """
            查找根节点
            :return:根节点列表
            """
            # self.root_node = list(filter(lambda x: x["father_id"] is None, data))
            for node in self.data:
                # 假定father_id是None就是根节点
                #例如有些数据库设计会直接把根节点标识出来
                if node["father_id"] is None:
                    self.root_node.append(node)
            return self.root_node

        def find_common_node(self) -> dict:
            """
            寻找共同的父节点
            :return: 共同的父节点字典
            """

            for node in self.data:
                father_id = node.get("father_id")
                # 排除根节点情况
                if father_id is not None:
                    # 如果父节点ID不在字典中则添加到字典中
                    if father_id not in self.common_node:
                        self.common_node[father_id] = list()
                    self.common_node[father_id].append(node)
            return self.common_node

        def build_tree(self, ) -> list:
            """
            生成目录树
            :return:
            """
            self.find_root_node()
            self.find_common_node()
            for root in self.root_node:
                # 生成字典
                base = dict(name=root["name"], id=root["id"], child=list())
                # 遍历查询子节点
                self.find_child(base["id"], base["child"])
                # 添加到列表
                self.tree.append(base)
            return self.tree

        def find_child(self, father_id: int, child_node: list):
            """
            查找子节点
            :param father_id:父级ID
            :param child_node: 父级孩子节点
            :return:
            """
            # 获取共同父节点字典的子节点数据
            child_list = self.common_node.get(father_id, [])
            for item in child_list:
                # 生成字典
                base = dict(name=item["name"], id=item["id"], child=list())
                # 遍历查询子节点
                self.find_child(item["id"], base["child"])
                # 添加到列表
                child_node.append(base)

    ```
- 测试
  ```python
    data = [
        {"id": 1, "father_id": None, "name": "01"},
        {"id": 2, "father_id": 1, "name": "0101"},
        {"id": 3, "father_id": 1, "name": "0102"},
        {"id": 4, "father_id": 1, "name": "0103"},
        {"id": 5, "father_id": 2, "name": "010101"},
        {"id": 6, "father_id": 2, "name": "010102"},
        {"id": 7, "father_id": 2, "name": "010103"},
        {"id": 8, "father_id": 3, "name": "010201"},
        {"id": 9, "father_id": 4, "name": "010301"},
        {"id": 10, "father_id": 9, "name": "01030101"},
        {"id": 11, "father_id": 9, "name": "01030102"},
    ]
    new_tree = Tree(data=data)

    print(new_tree.build_tree())
    # 输出
    # [{'name': '01', 'id': 1, 'child': [{'name': '0101', 'id': 2, 'child': [{'name': '010101', 'id': 5, 'child': []}, {'name': '010102', 'id': 6, 'child': []}, {'name': '010103', 'id': 7, 'child': []}]}, {'name': '0102', 'id': 3, 'child': [{'name': '010201', 'id': 8, 'child': []}]}, {'name': '0103', 'id': 4, 'child': [{'name': '010301', 'id': 9, 'child': [{'name': '01030101', 'id': 10, 'child': []}, {'name': '01030102', 'id': 11, 'child': []}]}]}]}]
  ```
## 总结
- 本文提供一种解决无限层级输出目录树的方法，还有其他更优的方案，仅供参考。
- 下一篇将介绍Flask实现微信Web端及APP端登录注册