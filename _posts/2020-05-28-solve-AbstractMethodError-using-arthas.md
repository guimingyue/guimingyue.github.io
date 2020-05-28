---
layout: post
title: 使用 Arthas 排查 AbstractMethodError
category: 其他
---


## 背景
本周在与团队同学合并代码分支测试时遇到一个问题 AbstractMethodError，记录下排查过程。业务上是一个 json 遍历的场景，使用的 json 库是 jackson，对于一个 `org.codehaus.jackson.JsonNode` 的实例对象（注意 jackson 中的 JsonNode 类是抽象类）调用其 `asText` 方法，然后就报了这个错误。
```java
java.lang.AbstractMethodError: org.codehaus.jackson.JsonNode.asText()Ljava/lang/String;
```
从错误信息上看，调用到的方法是一个抽象方法，所以错误就是 `AbstractMethodError` ，出现这种问题的原因基本上就是 jar 包冲突了。由于对平时一直使用 fastjson，对 jackson 也不怎么熟悉，并且 JsonNode 类是一个抽象类，其子类比较多，不能靠猜和试，所以就通过一些工具来确定真正的原因。
<a name="awB1y"></a>
## 使用 Arthas 排查
阿里巴巴开源的 [arthas](https://alibaba.github.io/arthas/) 是一个非常好用的工具，以前使用过来排查过一些问题，这次立马就想到了它。首先使用<br />Arthas 的 `sc` 命令看看加载到的 JsonNode 类和其所在的 jar 包，由于 sc 命令默认开启了子类匹配功能，所以所有当前类的子类也会被搜索出来，执行 sc 命令的部分结果如下（删掉不重要的信息）。
```java
sc -d org.codehaus.jackson.JsonNode

class-info        org.codehaus.jackson.node.TextNode
code-source       /path/to/war/WEB-INF/lib/jackson-mapper-asl-1.8.8.jar
name              org.codehaus.jackson.node.TextNode
isInterface       false
isAnnotation      false
isEnum            false
isAnonymousClass  false
isArray           false
isLocalClass      false
isMemberClass     false
isPrimitive       false
isSynthetic       false
simple-name       TextNode
modifier          final,public
annotation
interfaces
super-class       +-org.codehaus.jackson.node.ValueNode
                 +-org.codehaus.jackson.node.BaseJsonNode
                   +-org.codehaus.jackson.JsonNode
                     +-java.lang.Object
class-loader      +-org.apache.catalina.loader.ParallelWebappClassLoader
                 +-java.net.URLClassLoader@5c4xd650
                   +-sun.misc.Launcher$AppClassLoader@58x4vac2
                     +-sun.misc.Launcher$ExtClassLoader@3bgcf4f
```
从上面的信息中可以看到加载到的 jackson-mapper-asl 的版本是 1.8.8。知道了这个信息，其实再去看看使用的 jackson 中 JsonNode 所在的包的版本，其实就可以知道具体原因了。但是为了深挖到底，继续使用  `watch` 命令查看出错前方法调用的参数是什么？具体命令如下
```shell
watch package.a.b.c methodd "{params,returnObj}" -x 2 
```
通过该命令可以看到出错前方法调用的完整 json 参数，结合代码里的属性，可知引起报错的应该是一个 `org.codehaus.jackson.node.TextNode` 类的对象，而该类又是 JsonNode 的子类。而在 jackson-mapper-asl-1.8.8.jar 中，TextNode 类是不存在 `asText` 方法的，所以再调用该方法时就报了 `java.lang.AbstractMethodError` 这个错误。 再看看 JsonNode 所在的 jar 包的是 json-core-lgpl-1.9.13.jar，所以错误就非常明显了。<br />

<a name="ISdhF"></a>
## 总结
本文介绍了一种运行时遇到的 jar 包冲突的排查思路，使用 Arthas 确实很方便。其实排查 jar 问题的方法有很多，比如对于 maven 项目使用 `mvn dependency:tree` 命令打印出依赖版本依赖树，又比如在 IDEA 中有 maven helper 这样的插件，可以查看各种包的冲突版本。上文介绍的使用 Arthas 也是一种方法，适合在没有 IDE 工具或者 maven 工具的场景使用。

## Reference
* https://alibaba.github.io/arthas/
