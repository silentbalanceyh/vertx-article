实际上代码主体和lambda本身的逻辑在Vert.x并**不是同时执行**的。

```java
Metadata meta = new Metadata();
```

上边是构造了Metadata对象，它是在Vert.x中Verticle组件start方法调用时被调用，也就是deploy阶段。

```java
    boolean isSecure = meta.isSecure();
```

上边代码是执行`Metadata`对象的isSecure方法，位于lambda表达式内部，它是在请求触发时被调用——在Vert.x的Verticle组件执行deploy过程中，这段代码并不会执行（不仅这段，Handler内部代码都不会被执行），理解透这两个生命周期过后，就可以对上边代码进行改写了。

