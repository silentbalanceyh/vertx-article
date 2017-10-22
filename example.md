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
                        // 为空，则授权用户增加，使用AtomicInteger执行统计是因为多个线程共享，参考并发编程
                        if (count.incrementAndGet() == requiredcount) {
                            handler.handle(Future.succeededFuture());
                        }
                    } else if (sentFailure.compareAndSet(false, true)) {
                        // 出现异常，则抛出403的FORBIDDEN异常
                        handler.handle(Future.failedFuture(FORBIDDEN));
                    }
                } else {
                    handler.handle(Future.failedFuture(res.cause()));
                }

            };
            // 遍历每一个权限信息（authorities中存储了）
            Iterator var7 = this.authorities.iterator();

            while(var7.hasNext()) {
                String authority = (String)var7.next();
                if (!sentFailure.get()) {
                    // 针对每个权限信息调用User引用中的isAuthorised方法
                    user.isAuthorised(authority, authHandler);
                }
            }
        } else {
            handler.handle(Future.succeededFuture());
        }

    }
```



