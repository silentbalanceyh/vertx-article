既然已经找到了请求执行的入口，那么从入口开始分析BasicAuthHandlerImpl，让大家对整个请求流程有更加清晰的了解。为什么要分析？如果仅仅是针对开发人员，在处理过程开发自定义的`User`和`Provider`足够完成任务，了解清楚这部分内容的目的是方便大家“调试”，我们可以知道Vert.x究竟帮我们完成了什么事，这样从内到外抽丝剥茧，我们才知道我们开发的Provider和User最终是在整个认证授权框架的什么位置，以及如果我们要开发自定义的独立认证授权流程时从什么位置入手最符合我们实际项目的需要。先看下边的核心代码（方便大家理解，带上注释）

```java
    public void handle(RoutingContext ctx) {
        /**
         * 跨域访问中首次请求OPTIONS方法的判断逻辑，如果是OPTIONS则需要检查是否发送了请求头：
         * Access-Control-Request-Headers，如果包含了该请求头，那么需要检查对应的值中是否
         * 包含了认证需要的Authorization头信息，这个if判断描述了当前认证流程的入口条件。
         **/
        if (!this.handlePreflight(ctx)) {
            /**
             * 从RoutingContext中读取User对象
             **/
            User user = ctx.user();
            if (user != null) {
                /**
                 * 不解析请求头，直接从RoutingContext中拿到用户User执行认证
                 **/
                this.authorizeUser(ctx, user);
            } else {
                /**
                 * 无法读取User信息，一般是第一次请求
                 **/
                this.parseCredentials(ctx, (res) -> {
                    if (res.failed()) {
                        this.processException(ctx, res.cause());
                    } else {
                        User updatedUser = ctx.user();
                        if (updatedUser != null) {
                            Session session = ctx.session();
                            if (session != null) {
                                session.regenerateId();
                            }

                            this.authorizeUser(ctx, updatedUser);
                        } else {
                            this.getAuthProvider(ctx).authenticate((JsonObject)res.result(), (authN) -> {
                                if (authN.succeeded()) {
                                    User authenticated = (User)authN.result();
                                    ctx.setUser(authenticated);
                                    Session session = ctx.session();
                                    if (session != null) {
                                        session.regenerateId();
                                    }

                                    this.authorizeUser(ctx, authenticated);
                                } else {
                                    String header = this.authenticateHeader(ctx);
                                    if (header != null) {
                                        ctx.response().putHeader("WWW-Authenticate", header);
                                    }

                                    ctx.fail(401);
                                }

                            });
                        }
                    }
                });
            }
        }
    }
```



