# 基于Auth Common开发自定义认证组件

本文主要介绍vert.x中的认证授权流程，主要包含下边内容：

* 分析vertx-auth-common项目接口
* 以vertx-auth-oauth2为例分析项目实现方式
* 开发自定义的完整独立认证/授权模块
* 开发以vertx-grpc为基础的认证服务器（微服务架构可用）

## 术语表

* Handler：用于表达Vert.x Web中Router注册的Handler组件，该组件通常实现了接口`Handler<RoutingContext>`。

## 1. 楔子

提到系统中认证授权，通常就会想到RBAC（Role-Based Access Control）模型，该模型实际上描述了一个问题：Who, What, How——“Who对What进行了How的操作。”

* Who：权限拥有者（如Role、User、Group、Principal等）
* What：权限作用的主体对象，一般称为资源（Resource）
* How：具体的权限（Privilege）

本文不详细介绍该模型，细节参考：[https://en.wikipedia.org/wiki/Role-based\_access\_control](https://en.wikipedia.org/wiki/Role-based_access_control)，而本文主要对Vert.x中的认证授权部分进行详细介绍，尽可能分享在实际项目中用来处理认证授权部分的内容，官方地址：[http://vertx.io/docs/\#authentication\_and\_authorisation](http://vertx.io/docs/#authentication_and_authorisation)。

## 2. vertx-auth-common

Vert.x中主要包含了七个项目来处理认证授权的任务，vertx-auth-common是Vert.x中认证和授权的接口定义。在分析该项目之前，看看官方项目中Vert.x Web如何设置认证专用Handler的。

```java
AuthHandler basicAuthHandler = BasicAuthHandler.create(authProvider);

// All requests to paths starting with '/private/' will be protected
router.route("/private/*").handler(basicAuthHandler);
```



