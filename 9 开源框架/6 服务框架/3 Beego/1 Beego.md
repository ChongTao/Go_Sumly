# 一 Beego

**Beego** 是一个基于 Go 的 **全栈 Web 框架（Full-stack Framework）**，主要用于**快速开发后端 Web 服务和 API**。

核心特点：

- 全栈式 Web 框架
- 基于MCV结构
- 内置ORM
- 高性能

## 二 核心模块

- Router

  ```go
  beego.Router("/user/:id", &UserController{})
  
  // namespce：路由分组
  ns := beego.NewNamespace("/api",
      beego.NSRouter("/users", &UserController{}),
  )
  beego.AddNamespace(ns)
  ```

- Controller

  ```go
  type UserController struct {
      beego.Controller
  }
  ```

  每个HTTP请求创建一个Controller实例。

- ORM

  Beego自带ORM，支持Mysql、PostgreSQL

- 日志

  支持多种输出，console、file、syslog等