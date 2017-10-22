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

从上述代码段可以知道，最终调用的方法是`authorize`：

```java
    public void authorize(User user, Handler<AsyncResult<Void>> handler) {
        // 直接读取所需授权信息的数量（缓存过授权就会有该信息）
        int requiredcount = this.authorities.size();
        if (requiredcount > 0) {
            // 如果读取不到用户，则抛出FORBIDDEN的403异常信息
            if (user == null) {
                handler.handle(Future.failedFuture(FORBIDDEN));
                return;
            }
            AtomicInteger count = new AtomicInteger();
            AtomicBoolean sentFailure = new AtomicBoolean();
            // 执行一部授权检查，定义authHandler对象
            Handler<AsyncResult<Boolean>> authHandler = (res) -> {
                if (res.succeeded()) {
                    if (((Boolean)res.result()).booleanValue()) {
                        if (count.incrementAndGet() == requiredcount) {
                            handler.handle(Future.succeededFuture());
                        }
                    } else if (sentFailure.compareAndSet(false, true)) {
                        handler.handle(Future.failedFuture(FORBIDDEN));
                    }
                } else {
                    handler.handle(Future.failedFuture(res.cause()));
                }

            };
            Iterator var7 = this.authorities.iterator();

            while(var7.hasNext()) {
                String authority = (String)var7.next();
                if (!sentFailure.get()) {
                    user.isAuthorised(authority, authHandler);
                }
            }
        } else {
            handler.handle(Future.succeededFuture());
        }

    }
```



