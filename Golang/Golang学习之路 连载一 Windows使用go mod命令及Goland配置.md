# Golang学习之路 连载一 Windows使用go mod命令及Goland配置

tips:
- 本文使用的golang版本`go1.12.6`
- shell使用[Windows Terminal](https://github.com/microsoft/terminal)


## 一、安装Golang
- [官方网站下载](https://golang.org/dl/)
- [中文网站下载](https://studygolang.com/dl)


## 二、Golang项目目录结构
1. 使用GOPATH项目目录配置
    ``` 
    __
      |_ bin  # 编译后生成的可执行文件
      |_ pkg  # 编译后生成的文件
      |_ src  # 代码文件夹
        |_ tets_go_mod
    ```
2. 使用go mod管理目录配置
   ```
   __
     |_ tets_go_mod  # 存放源代码
   ```
## 三、`go mod`命令解释

|命令|含义|
|---|---|
|`go mod download`|下载依赖的module到本地cache|
|`go mod edit`|编辑go.mod文件|
|`go mod graph`|打印模块依赖图|
|`go mod init`|当前文件夹下初始化一个新的module, 创建go.mod文件|
|`go mod vendor`|将依赖复制到vendor下|
|`go mod tidy`|增加丢失的module，去掉未用的module|
|`go mod verify` |校验依赖|
|`go mod why` |解释为什么需要依赖|

## 四、`go mod`命令初始化一个项目
1. 新建项目目录`mkdir test_go_mod`并进入到项目目录`cd test_go_mod`
2. 开启`module`功能，执行命令`set GO111MODULE=on` 
3. 初始化项目`module`，执行命令`go mod init test_go_mod`
   ```
    tets_go_mod  # 代码文件夹
      |_ go.mod
   ```
4. 新建go文件`main.go`
   ```golang
   // main.go
    package main

    import "fmt"

    func main()  {
      fmt.Println("test_go_mod")
    }
   ```
   文件夹结构
   ```
    __
     |_ tets_go_mod  # 代码文件夹
         |_ go.mod
         |_ main.go
   ```

5. 运行`go run main.go`，控制台输出`test_go_mod`
 
6. 下载依赖，本文使用[gin](https://github.com/gin-gonic/gin)演示，`go get -u github.com/gin-gonic/gin`，修改`main.go`内容。
   ```golang
    package main

    import (
      "github.com/gin-gonic/gin"
    )

    func main() {
      r := gin.Default()
      r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{
          "message": "pong",
        })
      })
      _ = r.Run() // listen and serve on 0.0.0.0:8080
    }
   ```
   文件夹结构
   ```
    __
     |_ tets_go_mod  # 代码文件夹
         |_ go.mod
         |_ go.sum
         |_ main.go
   ```

7. 执行`go run main.go`
   - 报错`build command-line-arguments: cannot load github.com/gin-gonic/gin: open xxx: The system cannot find the path specified`
   - 需要先执行`go mod vendor`，将依赖复制到vendor下
   - 执行`go run main.go`，访问`http://localhost:8080/ping`，返回`{"message":"pong"}`


## 五、使用代理，解决某些包被墙

上述第六步下载包的时候，可能会出现某些依赖包无法下载的情况，被墙了。因此，网上主要有两种解决方法，一个是去[github](https://github.com/golang)克隆下载包到指定的位置，另外一直就是使用代理。本文使用代理解决包无法下载的问题，具体解决方法如下：
1. `golang`的版本要大于等于`1.11`
2. 使用`https://goproxy.io/`代理golang包下载。
3. 下载包之前开启`set GO111MODULE=on`,设置代理`set GOPROXY=https://goproxy.io`
4. 下载包即可，如果还是不行，建议重新安装最新版本的`golang`或者使用克隆的库的方法


## 六、完整流程及Goland配置
- 完整流程
  1. 新建项目目录并进入打开shell
  2. `set GO111MODULE=on`
  3. `set GOPROXY=https://goproxy.io`
  4. `go mod init test_go_mod` 
  5. `go get -u github.com/gin-gonic/gin`
  6. 编写`main.go`文件
      ```golang
      package main

      import (
        "github.com/gin-gonic/gin"
      )
      
      func main() {
        r := gin.Default()
        r.GET("/ping", func(c *gin.Context) {
          c.JSON(200, gin.H{
            "message": "pong",
          })
        })
        _ = r.Run() // listen and serve on 0.0.0.0:8080
      }
        ```
  7. `go mod vendor` 
  8. `go run main.go`
- Goland配置
 
  1. 找到  菜单栏 -> File-> Settings -> Go -> Go Modules(vgo) 选项
  2. 打开  Enable Go Modules(vgo) integration
  3. 打开  Vendoring mode
  4. 最后点击 apply 即可
  
## 总结
- 解决golang某些包无法下载问题
- Goland启用go module配置