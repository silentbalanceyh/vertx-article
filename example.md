### 2.4.总结

从源代码的分析看来，Vert.x中的认证和授权流程比较简单，而上边分析的BasicAuthHandler部分的内容是Vert.x Web项目的内容，并不是auth-common项目的内容，它们的整体结构图应该如下：

![](/assets/images/0001/01.png)仔细结合源代码分析上述结构图，实际上BasicAuthHandler

