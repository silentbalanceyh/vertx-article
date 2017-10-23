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

那么在实现自定义的认证授权框架时，先定义属于自己的User，这里分析我们自己使用的Basic认证，先定义几个核心属性：

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



