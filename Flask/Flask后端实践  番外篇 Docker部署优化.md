# Flask后端实践  番外篇 Docker部署优化

## 思考
[《Flask后端实践  连载八 Docker+Gunicorn+Nginx部署Flask后端》](https://blog.csdn.net/qq_22034353/article/details/8928940)一文中成功的部署了Flask应用。
1. 为啥占用的空间会那么大？可用`sudo docker ps -as | grep flask_test`查看。
2. 容器直接使用，flask应用产生的数据文件，例如日志、报表等文件哪里去了？

## 创建更小体积的容器
Q：为啥占用的空间会那么大？

A：`FROM python:3`拉取的是官方`python`镜像，包含`python`运行所需要的环境及其他辅助包，功能相当齐全。有些辅助包，实际`python`项目运行并不需要，那么我们只需要能用上的包就行。
### 构建alpine镜像
alpine镜像，简洁、小巧，基本是个空镜像。基于alpine的python镜像只构建了基础环境，[alpine-python镜像](https://github.com/jfloff/alpine-python)。
1. dockerfile
    - 修改之前
        ```dockerfile
        FROM python:3
        MAINTAINER "username<usereamil>"
        ENV PIPURL "https://pypi.tuna.tsinghua.edu.cn/simple"
        WORKDIR /projects

        COPY . .
        RUN pip --no-cache-dir install  -i ${PIPURL} --upgrade pip
        RUN pip --no-cache-dir install  -i ${PIPURL} gunicorn==19.9.0
        RUN pip --no-cache-dir install  -i ${PIPURL} flask==1.0.2
        RUN pip --no-cache-dir install  -i ${PIPURL} gevent==1.4.0

        CMD gunicorn  -c gun.conf app:app
        ```
    - 修改后
        ```dockerfile
        FROM python:3.7-alpine
        MAINTAINER "username<usereamil>"

        RUN echo http://mirrors.aliyun.com/alpine/v3.9/main > /etc/apk/repositories
        RUN echo http://mirrors.aliyun.com/alpine/v3.9/community >> /etc/apk/repositorie
       
        RUN apk update
        RUN apk --update add --no-cache gcc
        RUN apk --update add --no-cache g++

        ENV PIPURL "https://pypi.tuna.tsinghua.edu.cn/simple"
        WORKDIR /projects

        COPY . .
        RUN pip --no-cache-dir install  -i ${PIPURL} --upgrade pip
        RUN pip --no-cache-dir install  -i ${PIPURL} gunicorn==19.9.0
        RUN pip --no-cache-dir install  -i ${PIPURL} flask==1.0.2
        RUN pip --no-cache-dir install  -i ${PIPURL} gevent==1.4.0

        CMD gunicorn  -c gun.conf app:app
        ```
2. 构建镜像
    ```shell
    Sending build context to Docker daemon  4.608kB
    Step 1/15 : FROM python:3.7-alpine
    ---> 715a1f28828d
    Step 2/15 : MAINTAINER "username<usereamil>"
    ---> Using cache
    ---> b7bfa87a2b0e
    Step 3/15 : RUN echo http://mirrors.aliyun.com/alpine/v3.9/main > /etc/apk/repositories
    ---> Using cache
    ---> c82dd75ed328
    Step 4/15 : RUN echo http://mirrors.aliyun.com/alpine/v3.9/community >> /etc/apk/repositorie
    ---> Using cache
    ---> 7febd2453c8e
    Step 5/15 : RUN apk update
    ---> Using cache
    ---> 1f6b90e4471d
    Step 6/15 : RUN apk --update add --no-cache gcc
    ---> Using cache
    ---> ceafdbd4cd4f
    Step 7/15 : RUN apk --update add --no-cache g++
    ---> Running in a717db1c9018
    ---> d6149e90a126
    Step 8/15 : ENV PIPURL "https://pypi.tuna.tsinghua.edu.cn/simple"
    ---> Running in fef2f2a70fe5
    ---> b660c204a67a
    Step 9/15 : WORKDIR /projects
    ---> Running in b4f5e1625d78
    ---> b5412cc69410
    Step 10/15 : COPY . .
    ---> 969d2dba4cf2
    Step 11/15 : RUN pip --no-cache-dir install  -i ${PIPURL} --upgrade pip
    ---> Running in c608a175f3ee
    ---> 379af03e636d
    Step 12/15 : RUN pip --no-cache-dir install  -i ${PIPURL} gunicorn==19.9.0
    ---> Running in d585b4b53c76
    ---> 75da545b2456
    Step 13/15 : RUN pip --no-cache-dir install  -i ${PIPURL} flask==1.0.2
    ---> Running in 9f43837a308a
    ---> 5549e24cf9b4
    Step 14/15 : RUN pip --no-cache-dir install  -i ${PIPURL} gevent==1.4.0
    ---> Running in e12dabc30cfa
    ---> b7c026ddc0ba
    Step 15/15 : CMD gunicorn  -c gun.conf app:app
    ---> Running in 0eed5a480823
    Removing intermediate container 0eed5a480823
    ---> 530171c5de2b
    Successfully built 530171c5de2b
    Successfully tagged flask-test-alpine:latest
    ```
3. 运行
   - 执行命令运行`sudo docker run -it -p 8001:80 --name="flask_test_alpine" -d flask-test-alpine`
   - 查看是否启动`sudo docker ps `

4. 比较
   - 镜像执行命令`sudo docker images |grep flask`查询
        ```
        REPOSITORY          TAG                 IMAGE ID            SIZE
        flask-test-alpine   latest              530171c5de2b        288MB
        flask-test          latest              e90b8627afd0        965MB
        ```
    - 容器比较执行命令`sudo docker ps -as | grep flask_test`
        ```
        CONTAINER ID        IMAGE       ...  NAMES                SIZE
        5387acdc99f4  flask-test        ...  flask_test           822kB (virtual 966MB)
        9843d5822333  flask-test-alpine ...  flask_test_alpine    411kB (virtual 289MB)
        ```
    - 通过镜像和容器的比较，使用alpine容器和镜像都比原来的少一半以上，使用alpine为基础镜像的方法完胜。

## 挂载数据到宿主机
1. 数据卷

    数据卷是一个可以绕过联合文件系统的，专门指定的可在一或多个容器间共享目录。卷为提供为持久化或共享数据提供了一些有用的特性。 数据卷设计的初哀是提供持久化数据，而与容器的生命周期无关。因此，在删除容器时，Docker不会自动删除卷，直到没有容器再引用。如果需要在删除容器的同时移除数据卷。可以在删除容器的时候使用 命令`sudo docker rm -v [CONTAINER ID] `
    - 卷可以容器间共享和重用
    - 容器并不一定要和其它容器共享卷
    - 修改卷后会立即生效
    - 对卷的修改不会对镜像产生影响
    - 卷会一直存在，直到没有任何容器在使用它

2. 挂载
    
    使用命令`sudo docker -v 宿主机位置:容器位置`
    - 方法一，命令行
        ```shell
        sudo docker run -it -p 8001:80 --name="flask_test_alpine" -d  -v ./logs:/projects/logs flask-test-alpine
        ```
    - 方法二，docker-compose
        ```docker-compose
        version: "2"
        services:
            flask_test_alpine:
                image: flask-test-alpine:latest
                build: .
                container_name: flask_test_alpine
                restart: always
                ports:
                    - "8000:80"
                volumes:
                    - ./logs:/projects/logs             
        ```
   
## 总结
- 解决了docker部署占用空间过大的问题
- 解决了容器内文件与宿主机文件映射的问题

 