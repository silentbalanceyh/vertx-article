# 基于Auth Common开发自定义认证组件

本文主要介绍vert.x中的认证授权流程，主要包含下边内容：

* 分析vertx-auth-common项目
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

## 2. 深入vertx-auth-common

Vert.x中主要包含了七个项目来处理认证授权的任务，vertx-auth-common是Vert.x中认证和授权的接口定义。在分析该项目之前，看看官方项目中Vert.x Web如何设置认证专用Handler的。

```java
AuthHandler basicAuthHandler = BasicAuthHandler.create(authProvider);

// All requests to paths starting with '/private/' will be protected
router.route("/private/*").handler(basicAuthHandler);
```

如果上述代码第一次出现在你眼前，可能对于初学者有点不容易理解，那么本章节的目的就是解决初学者在Vert.x认证和授权过程中的困惑。Vert.x中对于认证和授权部分定义了几个核心接口，这些接口可以让开发人员创建属于自己的认证、授权逻辑。上述代码段中，使用`BasicAuthHandler`去创建了一个Handler组件，通过它创建的Handler组件和通常我们使用lambda表达式直接写的组件区别就在于引入了认证过程中的Provider接口，所以在路由内的代码执行流程中，Provider实现组件的逻辑会被触发，而Provider实现组件的逻辑就是开发人员需要真正关注的地方。vertx-auth-common中的几个核心接口（包括抽象类）如下：

* `io.vertx.ext.auth.User`：被认证的实体，包含了认证授权中该实体包含的所有数据信息。
* `io.vertx.ext.auth.AuthProvider`：认证专用接口。
* `io.vertx.ext.auth.AbstractUser`：实现了User接口的抽象类，抽象类的主体逻辑实现了简单的权限缓存和基本权限处理。

### 2.1. `BasicAuthHandler`做了什么？

由于Vert.x中的在Router处理Handler的过程中通常示例都是使用的lambda表达式的写法，很容易养成了一种固定写法的习惯，那么在了解`BasicAuthHandler`之前先看看Handler的两种写法，Vert.x内置的很多Handler使用的并不是lambda表达式的写法，而是直接定义，如：

```java
router.route().handler(CookieHandler.create());
router.route().handler(SessionHandler.create(LocalSessionStore.create(vertx)));
```

初学者有时候会好奇，它和下边的写法的区别在什么地方：

```java
router.route("/private/somepath").handler(routingContext -> {
  // This will have the value true
  boolean isAuthenticated = routingContext.user() != null;
});
```

本节主要对这两种写法进行剖析，最终使用哪种根据读者自己遇到的场景来定。先看一个改写的例子：

_（1）直接使用lambda表达式：_

```java
Metadata meta = new Metadata();
router.route("/api/*").handler(context -> {
    // 执行固定逻辑
    boolean isSecure = meta.isSecure();
    if(isSecure){
        // 执行额外逻辑
        // ......
    }else{
        context.next();
    }
});
```

上述代码实际上是一段伪代码，但它表示这样一段逻辑：在启动时，构造一个`Metadata`对象（称为元数据对象），并设置路由；在执行请求时，会调用meta的`isSecure`方法，它的返回结果会影响请求的执行流程。实际上代码主体和lambda本身的逻辑在Vert.x并**不是同时执行**的。

```java
Metadata meta = new Metadata();
```

上边是构造了Metadata对象，它是在Vert.x中Verticle组件start方法调用时被调用，也就是deploy阶段。

```java
    boolean isSecure = meta.isSecure();
```

上边代码是执行`Metadata`对象的isSecure方法，位于lambda表达式内部，它是在请求触发时被调用——在Vert.x的Verticle组件执行deploy过程中，这段代码并不会执行（不仅这段，Handler内部代码都不会被执行），理解透这两个生命周期过后，就可以对上边代码进行改写了。

_（2）使用Handler定义_

将（1）代码改写成下边这种模式：定义一个额外的类，`MetaHandler`用来创建Handler，并将`Metadata`对象的引用传给它。

主代码：

```java
Metadata meta = new Metadata();
MetaHandler handler = MetaHandler.create(meta);

router.route("/api/*").handler(handler);
```

Handler定义代码

```java
public class MetaHandler implements Handler<RoutingContext>{
    // 创建Handler的静态方法
    public static Handler<RoutingContext> create(final Metadata meta){
        return new MetaHandler(meta);
    }
    // 成员变量Metadata的对象引用
    private transient final Metadata reference;
    private MetaHandler(final Metadata reference){
        this.reference = reference;
    }
    @Override
    public void handle(final RoutingContext context){
        // 执行固定逻辑
        boolean isSecure = this.reference.isSecure();
        if(isSecure){
            // 执行额外逻辑
            // ......
        }else{
            context.next();
        }
    }
}
```

看看上边从读者最熟悉的lambda写法到Handler定义的改写，是不是就很清楚了`BasicAuthHandler`那段代码的作用了，它和第二种的主代码逻辑是一致的。

### 2.2. `BasicAuthHandler`背后的逻辑

Vert.x中的`BasicAuthHandler`远比上边的Handler定义部分复杂，接下来部分的内容对于实现自定义的认证授权很有帮助，但由于是分析Vert.x中的源代码，难免有些觉得枯燥。Vert.x中的`BasicAuthHandler`的代码定义如下：

```java
public interface BasicAuthHandler extends AuthHandler {
    String DEFAULT_REALM = "vertx-web";

    static AuthHandler create(AuthProvider authProvider) {
        return new BasicAuthHandlerImpl(authProvider, "vertx-web");
    }

    static AuthHandler create(AuthProvider authProvider, String realm) {
        return new BasicAuthHandlerImpl(authProvider, realm);
    }
}
```

实际上在调用`BasicAuthHandler`的create方法时，返回值是`AuthHandler`，而我们在Handler定义中返回的类型应该是一个`Handler<RoutingContext>`，实际上`AuthHandler`就是一个`Handler<RoutingContext>`的子接口：

```java
public interface AuthHandler extends Handler<RoutingContext> {
    @Fluent
    AuthHandler addAuthority(String var1);

    @Fluent
    AuthHandler addAuthorities(Set<String> var1);

    void parseCredentials(RoutingContext var1, Handler<AsyncResult<JsonObject>> var2);

    void authorize(User var1, Handler<AsyncResult<Void>> var2);
}
```

分析最初的代码和我们自己定义Handler部分的代码：

```java
AuthHandler basicAuthHandler = BasicAuthHandler.create(authProvider);
// 自己的定义
MetaHandler handler = MetaHandler.create(meta);
```

实际上上述两段代码最终都做了同样的事情，就是创建`Handler<RoutingContext>`对象，只有该对象会被Router识别，目前还没有进入实现部分，在我们的代码中，简单利用了`MetaHandler`处理了实现，而Vert.x中的`BasicAuthHandler`实现则是通过`BasicAuthHandlerImpl`类来完成的。从我们定义的Handler部分可以发现实现部分的代码是请求流程执行时触发的，它的主逻辑在于调用`handle(RoutingContext)`方法，上述代码的实现类`BasicAuthHandlerImpl`中似乎找不到？实际上Vert.x为了满足各种认证授权需求，进行了很细粒度的设计，它的整个继承树结构如：

```java
BasicAuthHandlerImpl extends AuthorizationAuthHandler {}

AuthorizationAuthHandler extends AuthHandlerImpl{}

AuthHandlerImpl implements AuthHandler{
    // ......
    public void handle(RoutingContext ctx){
        // 请求主代码逻辑
    }
}
```

 也就是说，真正在执行认证请求时候调用的是`AuthHandlerImpl`中的`handle(RoutingContext)`方法，那么到这里，请求怎么来的，相信读者就能够理解了。

