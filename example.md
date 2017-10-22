通过分析了认证和授权部分的源代码，就知道auth-common框架中的Provider和User的主方法在什么地方调用的：

* `Provider`：主要方法为`authenticate`方法，负责认证。
* `User`：主要方法为`isAuthorised`方法，负责授权。

至于Vert.x本身已经考虑了权限缓存、跨域OPTIONS的首次访问、开启用户会话的Session、以及默认情况下的403的处理等，当然这些内容细节读者可以参考Vert.x本身的源代码，这部分最后看看这两个主类的源代码（注释就不提供了）：

```java
// io.vertx.ext.auth.AuthProvider
@VertxGen
public interface AuthProvider {
  void authenticate(JsonObject authInfo, Handler<AsyncResult<User>> resultHandler);
}

// io.vertx.ext.auth.User
@VertxGen
public interface User {
  @Fluent
  User isAuthorized(String authority, Handler<AsyncResult<Boolean>> resultHandler);
  @Deprecated
  @Fluent
  default User isAuthorised(String authority, Handler<AsyncResult<Boolean>> resultHandler) {
    return isAuthorized(authority, resultHandler);
  }
  @Fluent
  User clearCache();
  JsonObject principal();
  void setAuthProvider(AuthProvider authProvider);
}
```



