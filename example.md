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
// 我们自己的定义

```



