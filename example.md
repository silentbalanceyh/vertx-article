实际上代码主体和lambda本身的逻辑在Vert.x并**不是同时执行**的。

```java
Metadata meta = new Metadata();
```

上边是构造了Metadata对象，它是在Vert.x中Verticle组件start方法调用时被触发，也就是deploy阶段。

```java

```



