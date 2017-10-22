上述代码段中三次访问到授权流程的代码，即代码中的方法`authorizeUser`，其实这个方法很简单，内容如下：

```java
    private void authorizeUser(RoutingContext ctx, User user) {
        this.authorize(user, (authZ) -> {
            if (authZ.failed()) {
                this.processException(ctx, authZ.cause());
            } else {
                ctx.next();
            }
        });
    }
```



