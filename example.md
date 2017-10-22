通过分析了认证和授权部分的源代码，就知道auth-common框架中的Provider和User的主方法在什么地方调用的：

* `Provider`：主要方法为`authenticate`方法，负责认证。
* `User`：主要方法为`isAuthorised`方法，负责授权。

至于Vert.x本身已经考虑了权限缓存，

