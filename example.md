通过分析了认证和授权部分的源代码，就知道auth-common框架中的Provider和User的主方法在什么地方调用的：

* `Provider`：主要方法为`authenticate`方法，负责认证。
* `User`：主要方法为`isAuthorised`方法，负责授权。

至于Vert.x本身已经考虑了权限缓存、跨域OPTIONS的首次访问、开启用户会话的Session、以及默认情况下的403的处理等，当然这些内容细节读者可以参考Vert.x本身的源代码，这部分最后看看这三个主类的源代码（注释就不提供了）：

`io.vertx.ext.auth.AuthProvider`

```java
@VertxGen
public interface AuthProvider {
    void authenticate(JsonObject authInfo, Handler<AsyncResult<User>> resultHandler);
}
```



