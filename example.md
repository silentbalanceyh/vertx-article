_（2）使用Handler分离定义_

将上述代码改写成下边这种模式：定义一个额外的类，`MetaHandler`用来创建Handler，并将`Metadata`对象的引用传给它。

主代码：

```java
Metadata meta = new Metadata();
router.route("/api/*").handler(MetaHandler.create(meta));
```

Handler定义代码

```java
public class MetaHandler implements Handler<RoutingContext>{
    public static Handler<RoutingContext> create(final Metadata meta){
        return new MetaHandler(meta);
    }
    private transient final Metadata reference;
    private MetaHandler(final Metadata reference){
        this.reference = reference;
    }
    @Override
    public void handle(final RoutingContext context){
        // 执行固定逻辑
        boolean isSecure = this.reference.isSecure();
        if(isSecure){
            // 执行额外逻辑
            // ......
        }else{
            context.next();
        }
    }
}
```



