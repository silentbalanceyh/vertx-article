# 基于Auth Common开发自定义认证组件

本文主要介绍vert.x中的认证授权流程，主要包含下边内容：

* 分析vertx-auth-common项目接口
* 以vertx-auth-oauth2为例分析项目实现方式
* 开发自定义的完整独立认证/授权模块
* 开发以vertx-grpc为基础的认证服务器（微服务架构可用）

## 1. 基础知识



