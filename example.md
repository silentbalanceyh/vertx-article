### 2.4.总结

从源代码的分析看来，Vert.x中的认证和授权流程比较简单，而上边分析的BasicAuthHandler部分的内容是Vert.x Web项目的内容，并不是auth-common项目的内容，它们的整体结构图应该如下：

![](/assets/images/0001/01.png)仔细结合源代码分析上述结构图，实际上`BasicAuthHandler`和`BasicAuthHandlerImpl`这两个类是vert.x web项目提供的内容，所有和认证授权相关的Handler部分都在`io.vertx.ext.web.handler.impl`包中，如果你觉得下边的头信息需要按照自己的逻辑进行解析，就不需要重写这两部分内容，否则的话，你也可以做深度定制，把这两个类重写。

```
[Basic认证]：Authorization: Basic XXXXXXX
[Digest认证]：Authorization: Digest realm="xxx", qop="auth", nonce="xxxx", opque="xxxx"
```

也就是说实现自定义的认证授权逻辑最简单的方式就是开发两个核心类，一个是AuthProvider（前文中提到的Provider），一个是User（实际上User的实现可以直接从AbstractUser继承，后边文章中会提到AbstractUser中的核心信息）。

