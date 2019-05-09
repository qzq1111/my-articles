# Docker+Gunicorn+Nginx部署Flask后端

tips:
- 本文主要介绍如何在docker中部署Flask APP
- [代码仓库](https://github.com/qzq1111/flask-resful-example)

## 背景
1. Flask自带的服务启动，非常方便在开发环境中调试使用，但是用于生产环境却不是好的选择。
2. 一般生产环境中部署Flask都是基于WGSI容器。
3. 生产环境可以用python的虚拟环境来部署Flask，但是部署方式比较麻烦，且不易移植。
## Gunicorn 
Gunicorn "绿色独角兽"是一个Python WSGI UNIX的HTTP服务器。这是一个预先叉工人模式，从Ruby的独角兽（Unicorn ）项目移植。Gunicorn服务器大致与各种Web框架兼容，只需非常简单的执行，轻量级的资源消耗，以及相当迅速。Gunicorn直接用命令启动，不需要编写配置文件，相对uWSGI要容易很多。[官网](https://gunicorn.org/)
## Docker
Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及应用运行的上下文环境到一个可移植的镜像中，然后发布到任何支持Docker的系统上运行。 通过容器技术，在几乎没有性能开销的情况下，Docker 为应用提供了一个隔离运行环境。
Docker有以下几个有点：
- 简化配置
- 代码流水线管理
- 提高开发效率
- 隔离应用
- 快速、持续部署

## Flask
### 一、测试Flask App
1. 服务器(ubuntu)中新建文件夹 `sudo mkdir /projects/FlaskTest`。
2. 进入文件夹 `cd /projects/FlaskTest`。
3. 新建文件`app.py`，写入内容。
    ```python
    from flask import Flask
    app = Flask(__name__)
    
    @app.route('/')
    def index():
        return 'hello world'
    
    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=80,)
    ```
4. 新建文件`gun.conf `
    ```conf
    # 并行工作进程数
    workers = 2
    # 指定每个工作者的线程数
    threads = 4
    # 监听内网端口80
    bind = '0.0.0.0:80'
    # 工作模式协程
    worker_class = 'gevent'
    # 设置最大并发量
    worker_connections = 2000
    # 设置进程文件目录
    pidfile = 'gunicorn.pid'
    # 设置访问日志和错误信息日志路径
    accesslog = 'gunicorn_acess.log'
    errorlog  = 'gunicorn_error.log'
    # 设置日志记录水平
    loglevel = 'info'
    # 代码发生变化是否自动重启
    reload=True
    ```
### 二、编写Dockerfile
在`FlaskTest`目录中新建`Dockerfile`文件，写入内容。
```dockerfile
FROM python:3
MAINTAINER "username<usereamil>"
ENV PIPURL "https://pypi.tuna.tsinghua.edu.cn/simple"
WORKDIR /projects

COPY app.py app.py
RUN pip --no-cache-dir install  -i ${PIPURL} --upgrade pip
RUN pip --no-cache-dir install  -i ${PIPURL} gunicorn==19.9.0
RUN pip --no-cache-dir install  -i ${PIPURL} flask==1.0.2
RUN pip --no-cache-dir install  -i ${PIPURL} gevent==1.4.0

CMD gunicorn  -c gun.conf app:app
```
#### 说明
Dockerfile 文件是用于定义 Docker 镜像生成流程的配置文件，文件内容是一条条指令，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建；这些指令应用于基础镜像并最终创建一个新的镜像，可以认为用于快速创建自定义的Docker镜像。
#### 1.FORM
指定基础镜像（必须有的指令，并且必须是第一条指令）

#### 2.MAINTAINER
用于提供信息的指令，用于让作者提供本人的信息；不限制其出现的位置，但建议紧跟在FROM之后

格式：
```
MAINTAINER <name>
```
#### 3.ENV
设置环境变量，可以在RUN之前使用，然后RUN命令时调用，容器启动时这些环境变量都会被指定

格式：
```
ENV <key> <value>  一次定义一个变量
ENV <key>=<value> ...   一次可定义多个变量 
```

#### 4.WORKDIR
WORKDIR指令可以来指定工作目录（或者称为当前目录），以后各层的当前目录就被改为指定的目录，如果目录不存在，WORKDIR会建立目录

格式：
```
WORKDIR <工作目录路径>
```
#### 5.COPY
格式：
```
COPY <源路径>... <目标路径>
COPY ["<源路径1>",... "<目标路径>"]
```
COPY 指令将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置

#### 6.RUN
用于执行命令行命令

格式：
```
RUN <命令>
```

#### 7.CMD
类似于RUN指令，用于运行程序；但二者运行的时间点不同；CMD在docker run时运行，而非docker build

### 三、构建镜像
FlaskTest目录下执行 `sudo docker build . -t=flask-test:latest`命令，该命令作用是创建/构建镜像，`-t` 指定名称及版本号`flask-test:latest`，点 `.` 构建内容为当前上下文目录。大致执行内容如下输出
```
Sending build context to Docker daemon  4.096kB
Step 1/10 : FROM python:3
...
Step 2/10 : MAINTAINER "username<usereamil>"
...
Step 3/10 : ENV PIPURL "https://pypi.tuna.tsinghua.edu.cn/simple"
...
Step 4/10 : WORKDIR /projects
...
Step 5/10 : COPY . .
...
Step 6/10 : RUN pip --no-cache-dir install  -i ${PIPURL} --upgrade pip
...
Step 7/10 : RUN pip --no-cache-dir install  -i ${PIPURL} gunicorn==19.9.0
...
Step 8/10 : RUN pip --no-cache-dir install  -i ${PIPURL} flask==1.0.2
...
Step 9/10 : RUN pip --no-cache-dir install  -i ${PIPURL} gevent==1.4.0
...
Step 10/10 : CMD gunicorn  -c gun.conf wsgi_gunicorn:app
...
Successfully built 277d7a1a9f4e
Successfully tagged flask-test:latest
```
### 四、查看镜像
控制台执行命令`sudo docker images`查看所有的镜像，确认构建的 `flask-test:latest` 镜像是否存在
```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
flask-test          latest              277d7a1a9f4e        13 minutes ago      965MB
python              3                   59a8c21b72d4        2 weeks ago         929MB
...
```
### 五、创建并启动容器
1. 控制台执行命令`sudo docker run -d -p 8000:80 flask-test:latest`
2. 控制台执行`sudo curl http://127.0.0.1:8000`  返回`hello world`
3. 控制台执行`sudo docker ps `查看启动的容器
    ```
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
    47e55887d773        flask-test:latest   "/bin/sh -c 'gunicor…"   2 minutes ago       Up 2 minutes        0.0.0.0:8000->80/tcp                unruffled_zhukovsky
    ```
4. 说明
   - `-d`后台守护模式
   - `-p `指定容器与宿主机端口映射
### 六、更优雅的启动容器，使用docker-compose
1. 编写`docker-compose.yaml`文件
    ```docker-compose
    version: "2"
    services:
        flask_test:
            image: flask-test:latest
            build: .
            container_name: flask_test
            restart: always
            ports:
                - "8000:80"
    ```
2. 控制台执行命令`docker-compose up -d`启动容器
3. 控制台执行命令`sudo docker ps `验证容器`flask_test`是否启动
4. 说明
    - version：docker-compose的版本
    - services：需要管理的服务
    - flask_test：FlaskApp服务名称
    - image：FlaskApp服务镜像来源
    - build：如果镜像不存在，当前位置构建镜像。存在则跳过
    - container_name：启动的容器名称
    - restart：容器启动属性， always一直重启
    - ports：容器与宿主机的端口映射

## Nginx配置
1. 安装好nginx之后，执行命令`cd /etc/nginx/conf.d` 进入到`conf.d`目录
2. 新建`flask_test.conf`文件并填入以下内容
    ```conf
    server {
        listen       80;
        server_name  www.qinzhiqian.xyz;

        # api代理转发
        location / {
            proxy_redirect  off;
            proxy_set_header    Host $host;
            proxy_set_header    X-Real-IP            $remote_addr;
            proxy_set_header    X-Forwarded-For      $proxy_add_x_forwarded_for;
            proxy_set_header    X-Forwarded-Proto    $scheme;
            proxy_pass http://127.0.0.1:8000;
        }
    }
    ```
3. 重启nginx执行命令 `sudo service nginx reload`
4. 外网访问配置好的url即可，例如：http://www.qinzhiqian.xyz


## 总结
- 本文将Flask部署在docker容器中，Flask优化部署相关的内容参考[番外篇  Docker部署优化](https://blog.csdn.net/qq_22034353/article/details/89950228)
- 下一篇文章将介绍Flask定时任务
