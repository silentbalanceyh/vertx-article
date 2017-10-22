_（2）使用Handler分离定义_

将上述代码改写成下边这种模式：定义一个额外的类，MetaHandler用来创建Handler，并将Metadata对象的引用传给它。

