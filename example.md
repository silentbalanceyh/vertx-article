## 3. 实现自定义认证授权流程

从第二节的分析中，我们已经知道了Vert.x中提供的认证授权框架的流程，那么本章节实现一个定制力度比较大的认证授权框架，一方面加深读者对第二节的理解，另外一方面让大家对认证授权的各种开发更加耳熟能详。实际上我们在项目开发过程中重写的目的有几点：

* 重新定义了Token接口，解析流程参考了上边的parseCredentials方法；
* 启用了OSGI插件模型，将所有的认证授权部分全部放到插件中去完成；
* 参考Provider/User/Handler这几个结构，定义了新的CommonToken的基础模型来完成认证授权的重新定义；
* 强化授权流程，连接自己的RBAC模型实现授权管理；

### 3.1.分析AbstractUser

Vert.x默认提供的AbstractUser可以称为是被认证授权的实体，主要包含几个核心内容：

* 实现了isAuthorized方法用于授权（授权模型使用`Set<String>`类型的权限集合）；
* 实现了doIsPermitted用于核心授权检查逻辑，和isAuthorized方法配合完成；
* 实现了ClusterSerializable接口，用于在Vert.x的Cluster环境实现快速序列化和反序列化的操作；

那么在实现自定义的认证授权框架时，先定义属于自己的User，该User的定义如下：

```java
public class BasicUser extends AbstractUser
```

这里分析一下我们自己使用的Basic认证BasicUser类，定义的核心属性：

```java
    /**
     * User中需要使用的Provider对象引用
     **/
    @SuppressWarnings("unused")
    private transient AuthProvider provider;
    /**
     * 用户名信息
     **/
    private transient String username;
    /**
     * 用户Id信息
     **/
    private transient String id;
    /**
     * 用户的Password加密字符串
     **/
    private transient String password;
    /**
     * 用户授权信息
     **/
    private transient JsonObject principal;
```

这里扩展了原生的username和password两个字段，添加了id（数据库代理主键）和JsonObject类型的principal，一般principal存储了一些附加的数据（比如当前用户的角色、基本权限、资源集合等，这个根据实际需要而有所不同）。默认情况下如果要User对象在集群模式中可使用（应该说所有需要在集群模式中使用的数据对象），都会实现`io.vertx.core.shareddata.impl.ClusterSerializable`接口，AbstractUser本身实现了该接口，所以需要针对接口写类似下边的方法：

```java
     /** **/
    @Override
    public void writeToBuffer(final Buffer buffer) {
        /** 1.调用父类方法 **/
        super.writeToBuffer(buffer);
        /** 2.写入id，用户名和token **/
        Buffalo.write(buffer, id, username, password);
    }

    /** **/
    @Override
    public int readFromBuffer(int pos, final Buffer buffer) {
        /** 1.从父类读取 **/
        pos = super.readFromBuffer(pos, buffer);
        /** 2.读取信息 **/
        final String[] reference = new String[3];
        pos = Buffalo.read(pos, buffer, reference);
        /** 3.从引用中读取数据 **/
        this.id = reference[0];
        this.username = reference[1];
        this.password = reference[2];
        return pos;
    }
```

这两个方法只有一个地方要注意，就是顺序，上边代码封装了读写Buffer的两个方法，两个方法的实现如下：

```java
    public static void write(@NotNull final Buffer buffer,
                             final String... data) {
        // 遍历数据
        for (final String item : data) {
            if (StringKit.isNonNil(item)) {
                // 字节数据
                final byte[] bytes = item.getBytes(Resources.ENCODING);
                buffer.appendInt(bytes.length);
                buffer.appendBytes(bytes);
            }
        }
    }

    public static int read(final int start,
                           @NotNull final Buffer buffer,
                           @NotNull final String[] reference) {
        int pos = start;
        for (int idx = 0; idx < reference.length; idx++) {
            // 先读取长度信息
            final int len = buffer.getInt(pos);
            // 计算偏移量
            pos += 4;
            // 读取本身内容
            final byte[] bytes = buffer.getBytes(pos, pos + len);
            reference[idx] = new String(bytes, Resources.ENCODING);
            pos += len;
        }
        return pos;
    }
```

从上边代码可以看到，writeToBuffer的顺序是`id, username, password`，那么在readFromBuffer的过程中，其读取的顺序也是`id, username, password`，对于Buffer类型的细节这里不讲，主要是提醒一下大家，如果定义User，必定会重写这两个方法，个人的建议就是把能够标识用户身份的数据写入到Buffer中（如果这些数据需要执行权限检查，那么也需要写入进入。）写入的核心点就在于属性字段的顺序，一定要小心（这是序列化和反序列化的操作）。

最后需要说明的是AbstractUser已经实现了默认的isAuthorized方法，该方法的核心实现如下：

```java
    public User isAuthorized(String authority, Handler<AsyncResult<Boolean>> resultHandler) {
        if (this.cachedPermissions.contains(authority)) {
            resultHandler.handle(Future.succeededFuture(true));
        } else {
            this.doIsPermitted(authority, (res) -> {
                if (res.succeeded() && ((Boolean)res.result()).booleanValue()) {
                    this.cachedPermissions.add(authority);
                }

                resultHandler.handle(res);
            });
        }
        return this;
    }
```

实际上该方法会调用doIsPermitted方法用来检查权限信息，而AbstractUser开启了权限缓存，那么我们在重写过程中，仅仅需要实现doIsPermitted方法就足够了。

```java
    /**
     * 权限检查
     **/
    @Override
    public void doIsPermitted(final String permission, 
                              final Handler<AsyncResult<Boolean>> resultHandler) {
        // ....权限检查的核心逻辑
    }
```



