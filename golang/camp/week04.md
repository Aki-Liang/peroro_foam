# 第四周

## 工程项目结构

### Standard Go Project Layout

    开发工具创建目录，来推动规范

    
####  /cmd

```
|-cmd
|  |
|  |-demo1
|  |   |-main.go
|  |-demo2
|  |   |-main.go   
```

每个应用程序的目录名应该与想要的可执行文件的名称相匹配

cmd目录中不要放太多代码

#### /internal

存放所有的私有的应用程序和库代码，不希望其他人在其应用程序或库中导入的代码

* 在编译层进行隔离，参考Go 1.4 release notes

#### /pkg

```
 |-pkg
 |  |-cache
 |  |   |-redis
 |  |   |-memcache
 |  |-conf
 |  |   |-dsn
 |  |   |-env
```

存放外部应用程序可以使用的库代码，其他项目会导入这些库。

### Kit Project Layout （基础库）

Package Origented Design

Kit项目必须具备的特点
* 统一
* 标准库方式布局
* 高度抽象
* 支持插件

Kit库不应使用vendor

package命名方法 //todo

### Service Application Project Layout

```
|-README.md
|-api
|-cmd
|-configs
|-go.mode
|-go.sum
|-internal
|-test
```

* /api
  * API协议定义目录，protobuf文件，以及生成的go文件
* /configs
  * 配置文件模板或默认配置
* /test
  * 外部测试应用程序和测试数据


微服务需要一个全局唯一的名称

公司需要维护一个服务树，管理应用名称

三段式命名,按业务来命名


## API 设计

## 配置管理

## 包管理

## 测试

## 引用
