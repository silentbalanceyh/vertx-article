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



