# Python基于Drone的CI-CD（代码检查、测试、构建、部署）实践

tips:
- 全流程实践CI/CD
- [实验代码代码仓库](https://github.com/qzq1111/flask-resful-example)

## 一、CI/CD 服务器环境配置

### 1. CI/CD 工具列表
- 容器服务：安装 docker、docker-compose
- Drone服务器： 安装配置 drone
- 代码检查服务器：安装配置 SonarQube


### 2. 基于Docker的Drone和SonarQube安装配置

`docker-compose.yaml`配置如下：
```yaml
version: "2"
services:
  drone-server:
    image: drone/drone:1
    container_name: drone_server
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data/drone:/data
    environment:
      - DRONE_GITHUB_SERVER=https://github.com
      - DRONE_RUNNER_CAPACITY=2
      - DRONE_TLS_AUTOCERT=true
      - DRONE_SERVER_PROTO=http
      # 此三项需要修改为你自己的Github应用申请
      # 具体参考文档 https://docs.drone.io/installation/github/single-machine/
      - DRONE_GITHUB_CLIENT_ID=4985615
      - DRONE_GITHUB_CLIENT_SECRET=132456 
      - DRONE_SERVER_HOST=drone.xxxx.com
      
    ports:
      - "8001:80"
      - "8000:443"
    networks: 
      - ci_net
  sonarqube-service:
    image: sonarqube:alpine
    container_name: sonarqube_service
    restart: always
    ports:
      - "8002:9000"
    networks: 
      - ci_net

networks: 
  ci_net:
```

字段说明：

|字段|说明|
|---|---|
|DRONE_GITHUB_SERVER|Github版本库访问地址|
|DRONE_RUNNER_CAPACITY|一个整数，定义代理程序应同时执行的最大管道数。默认值为两个管道。|
|DRONE_SERVER_HOST|包含Drone服务器主机名或IP地址的字符串。|
|DRONE_SERVER_PROTO|包含Drone服务器协议方案的字符串。该值应设置为http或https|
|DRONE_GITHUB_CLIENT_ID|Github中申请的应用连接ID |
|DRONE_GITHUB_CLIENT_SECRET|Github中申请的应用连接密钥 |
|DRONE_SERVER_HOST| Github中申请应用填写的drone服务地址 |



开启Drone和SonarQube服务：
- 执行`docker-compose up -d`运行drone
- 执行`docker ps -a`查看是否启动
- 如果未成功启动，执行`docker logs [容器ID]`查看错误并解决。

访问服务：
- 访问服务器**8001**端口，打开drone的web页面，使用**Github账号**登陆，登陆成功drone会自动获取当前用户项目，点击项目开启对项目的监管。
- 访问服务器**8002**端口，打开SonarQube的web界面，默认账户  **admin**  默认密码 **admin**。

参考文档：
- [基于GitHub版本控制库Drone安装和配置文档](https://docs.drone.io/installation/github/single-machine/)
- [更多Drone安装和配置版本库安装方法](https://docs.drone.io/installation/)
- [SonarQube安装配置文档](https://hub.docker.com/_/sonarqube/)

## 二、项目代码中的CI/CD配置

### 1. 注意事项

- Github代码仓库中需要配置webhook，在登陆drone后，会自动拉取项目，点击进入项目后，点击settings后，进入到设置界面，然后点击save，即可完成webhook配置（**drone自动配置**）。
- 自动完成了Github的webhook配置，注意检查协议，应与drone地址协议一致（**http或https**）。
- from_secret值在 **drone/settings/Secrets**配置。


### 2. 编写`.drone.yml`文件
- 每一个项目都需要编写`.drone.yml`文件，在文件中定义代码提交了之后，执行具体的操作步骤。
- 在实际项目中会涉及到各种插件的使用，根据需要在[drone插件库](http://plugins.drone.io/)查询相关插件的使用方法。

- 以Python项目为例，编写`.drone.yml`：
  ```yaml
  kind: pipeline
  name: default

  steps:
  # 代码分析
  - name: code-analysis
    image: aosapps/drone-sonar-plugin
    settings:
      sonar_host:
        from_secret: sonar_host
      sonar_token:
        from_secret: sonar_token

  # 测试
  - name: code-test
    image: python:3
    commands:
    - pip install -r requirements.txt
    - pytest

  # 构建代码docker镜像
  - name: code-build
    image: plugins/docker
    settings:
      registry: 
        from_secret: docker_registry
      repo: 
        from_secret: docker_repo
      username: 
        from_secret: docker_name
      password: 
        from_secret: docker_password
      tags:
        - latest
        - '1.0'

  # 将部署文件放入指定目录
  - name: code-scp
    image: appleboy/drone-scp
    settings:
      host: 
        from_secret: ssh_host
      username: 
        from_secret: ssh_user
      port: 22
      password: 
        from_secret: ssh_password
      target: /projects/ci_test
      source: docker-compose.yaml

  # 部署项目
  - name: code-deploy
    image: appleboy/drone-ssh
    settings:
      host: 
        from_secret: ssh_host
      username: 
        from_secret: ssh_user
      port: 22
      password: 
        from_secret: ssh_password
      script:
        - cd /projects/ci_test
        - docker-compose pull ci_test
        - docker-compose up -d
  ```

## 三、配置说明

### 1. 代码分析
- [配置参考文档](http://plugins.drone.io/aosapps/drone-sonar-plugin/)
- 使用`sonarQube`插件进行代码质量分析，在`sonarQube`中项目名字默认为仓库名称
```yaml
kind: pipeline
name: default

steps:
# 代码分析
- name: code-analysis
  image: aosapps/drone-sonar-plugin
  settings:
    sonar_host:
      from_secret: sonar_host
    sonar_token:
      from_secret: sonar_token
```
字段说明：
| 字段|说明|
|---|---|
|sonar_host|sonar代码分析后推送地址|
|sonar_token|sonar用户认证密钥，如果未设置，不需要填|

### 2. 测试
以python为例。
```yaml
kind: pipeline
name: default

steps:
# 测试
- name: code-test
  image: python:3
  commands:
  - pip install -r requirements.txt
  - pytest
```


### 3. 构建镜像
- [配置参考文档](http://plugins.drone.io/drone-plugins/drone-docker/)
- 构建docker镜像并推送到远端镜像仓库，需要在代码仓库项目根目录编写`Dockerfile`文件
```yaml
kind: pipeline
name: default

steps:
# 构建代码docker镜像
- name: code-build
  image: plugins/docker
  settings:
    registry: 
      from_secret: docker_registry
    repo: 
      from_secret: docker_repo
    username: 
      from_secret: docker_name
    password: 
      from_secret: docker_password
    tags:
      - latest
      - '1.0'
```
字段说明：
| 字段|说明|
|---|---|
|registry|镜像推送地址|
|repo|推送的命名空间及镜像名字（仓库地址/命名空间/镜像名称）|
|username|仓库账号|
|password|仓库账号密码|
|tags|设置镜像标签或版本|


### 4. 放置部署文件
- [配置参考文档](http://plugins.drone.io/appleboy/drone-scp/)
- 将代码仓库中的`docker-compose.yaml`文件放置到指定服务器的指定位置。
```yaml
kind: pipeline
name: default

# 将部署文件放入指定目录
- name: code-scp
  image: appleboy/drone-scp
  settings:
    host: 
      from_secret: ssh_host
    username: 
      from_secret: ssh_user
    port: 22
    password: 
      from_secret: ssh_password
    target: /projects/ci_test
    source: docker-compose.yaml
```
字段说明：
| 字段|说明|
|---|---|
|host|服务器地址|
|username|服务器用户名|
|password|服务器用户密码|
|target|指定放置的文件路径|
|source|选用放置的文件路径|

### 5. 部署项目
- [配置参考文档](http://plugins.drone.io/appleboy/drone-ssh/)
- 指定的服务器部署项目
```yaml
kind: pipeline
name: default

# 部署项目
- name: code-deploy
  image: appleboy/drone-ssh
  settings:
    host: 
      from_secret: ssh_host
    username: 
      from_secret: ssh_user
    port: 22
    password: 
      from_secret: ssh_password
    script:
      - cd /projects/ci_test
      - docker-compose pull ci_test
      - docker-compose up -d
```
字段说明：
| 字段|说明|
|---|---|
|host|服务器地址|
|username|服务器用户名|
|password|服务器用户密码|
|script|服务器中的执行命令|

### 6. 按照Git条件来执行
- [配置参考文档](https://docs.drone.io/user-guide/pipeline/conditions/)
- 下面的示例中，步骤仅限于`master`以下`tag`事件 
```yaml
kind: pipeline
name: default

steps:
# 代码分析
- name: code-analysis
  image: aosapps/drone-sonar-plugin
  settings:
    sonar_host:
      from_secret: sonar_host
    sonar_token:
      from_secret: sonar_token
  when:
     branch:
      - master
     event:
      - tag
```
