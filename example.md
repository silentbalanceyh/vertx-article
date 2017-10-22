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
                // 不解析请求头，直接从RoutingContext中拿到用户User执行认证
                this.authorizeUser(ctx, user);
            } else {
                /**
                 * 无法读取User信息，直接解析Http的头：Authorization，一般是第一次请求时调用
                 * 注意：parseCredentials方法是定义于AuthHandler中的
                 * 被解析的值格式一般是：Authorization: Type XXXXXX，这里的Type只能是下边的值：
                 * Basic, Digest, Bearer, HOBA, Mutual, Negotiate, OAuth, SCRAM-SHA-1, SCRAM-SHA-256
                 * 一般这个方法会被重写，不同类型的值解析逻辑会不同，BasicAuthHandlerImpl中的
                 * parseCredentials方法就被重写过，主要用于解析Basic中的头信息。
                 **/
                this.parseCredentials(ctx, (res) -> {
                    if (res.failed()) {
                        // 解析失败，直接报错
                        this.processException(ctx, res.cause());
                    } else {
                        // 解析成功，执行二次逻辑，判断Session中是否有当前用户
                        User updatedUser = ctx.user();
                        if (updatedUser != null) {
                            Session session = ctx.session();
                            if (session != null) {
                                session.regenerateId();
                            }
                            // 当前用户已经登陆过了，直接使用Session中的User对象执行验证
                            this.authorizeUser(ctx, updatedUser);
                        } else {
                            /**
                             * 从上述代码逻辑可以知道，直到主流程运行到这个位置，代码才真正到达Provider中的，而这里就
                             * 会调用getAuthProvider方法，从当前系统中读取已定义过的Provider，并且调用provider的
                             * 主逻辑authenticate方法。那么读者也许比较困惑的是res.result()返回的应该是什么，这里
                             * 返回的内容实际上是由上层调用parseCredentials来决定的，一般是一个JsonObject对象，但
                             * 具体数据由不同的Handler实现来决定，比如Basic类型的最后一行为：
                             * handler.handle(Future.succeededFuture(
                             *     (new JsonObject()).put("username", suser).put("password", spass))
                             * )
                             * 那么它返回的就是一个JsonObject对象，形如：
                             * {
                             *     "username":"xxxx"
                             *     "password":"xxxx"
                             * }
                             * 这也是官方教程中提到Basic的数据格式的原因，需要再提到的一点就是Provider接口会将一个
                             * JsonObject类型的对象转换成User被认证的实体对象，所以最终从authenticate第二参数返回
                             * 的数据类型就是User。
                             **/
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
                                    /**
                                     * authenticateHeader一般又是一个会被子类重写的方法，它用于设置认证不成功时
                                     * 在响应中提供WWW-Authenticate的头信息，对于Basic而言一般是Basic realm=xxx
                                     * 格式。
                                     **/
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



