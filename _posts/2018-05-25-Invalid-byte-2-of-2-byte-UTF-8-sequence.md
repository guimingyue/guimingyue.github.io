---
layout: post
title: Invalid byte 2 of 2-byte UTF-8 sequence
category: Java
---
前几天在使用 Spring 解析一个 XML bean 文件时遇到了编码的问题，报错异常栈如下图所示。
```java


```
关键的报错信息时`Invalid byte 2 of 2-byte UTF-8 sequence.`这个错误，很多文章提到的解决办法是将 encoding 改为 GBK 编码或者其他编码，但是为什么呢？产生这个问题的根本原因是什么？  
为了搞清楚这个问题，我们先简化代码，关键代码如下。
```java
String data = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n"
            + "<!DOCTYPE beans PUBLIC \"-//SPRING//DTD BEAN//EN\" \"http://www.springframework.org/dtd/spring-beans"
            + ".dtd\">\n"
            + "<beans>\n"
            + "  <bean class=\"me.guimy.Student\" id=\"student\">\n"
            + "    <property name=\"name\">\n"
            + "      <value><![CDATA[中文]]></value>\n"
            + "    </property>\n"
            + "  </bean>\n"
            + "</beans>";
```
这段代码中加载一个 Spring 的 XML bean 文件，其中有一个值是中文，很显然在解析到中文的时候就报了`Invalid byte 2 of 2-byte UTF-8 sequence.`错。可以看到已经定义了 `encoding="UTF-8"`，所以理论上解析在解析的时候不应该无法解析中文的错误。难道是遇到 JDK 的 bug 了？从异常栈可以看到 Spring 在解析 XML 调用了 JDK 中`com.sun.org.apache.xerces.internal.jaxp`包下的类来解析，所以针对这个疑问，可以先 debug 到 具体的类中排查到底使用什么编码在解析，其实也可以从异常栈的`UTF8Reader`可以看到是使用 UTF-8 的编码在解析。通过 debug 到`UTF8Reader`可以了解到，这个程序执行过程中解析 XML 的过程其实是解析的文本对应的字节序列。  
既然知道了解析 XML 文本对应的字节序列是使用的 UTF-8 编码，并且 XML 文本中也指定了 UTF-8 编码，那么怀疑的方向就是字节序列是怎么生成的，是否是 UTF-8 编码对应的字节序列？所以现在需要确定的是如何将 XML 文本转换成字节序列的。  
可以看看 Spring 的`StringXmlApplicationContext`这个类的构造方法。 
```java
public StringXmlApplicationContext(String[] stringXmls, ApplicationContext parent, ClassLoader cl) {
        
    }
```
传入的参数是一个字符串数组，然后遍历每个字符串对象，这段代码中 `stringXmls[i].getBytes()` 将字符串转换为字节数组使用了默认的编码，再看`String.getBytes`方法的代码，
```java
public byte[] getBytes() {
    return StringCoding.encode(value, 0, value.length);
}
static byte[] encode(char[] ca, int off, int len) {
    String csn = Charset.defaultCharset().name();
    try {
        return encode(csn, ca, off, len);
    } catch (UnsupportedEncodingException x) {
        warnUnsupportedCharset(csn);
    }
    try {
        return encode("ISO-8859-1", ca, off, len);
    } catch (UnsupportedEncodingException x) {
        System.exit(1);
    	return null;
    }
}
```
从`String csn = Charset.defaultCharset().name()`这行代码可以看到去拿了默认的编码，而默认的编码是通过`file.encoding`来指定的，所以很简单，打印一下`Syste.getProperty("file.encoding")`的值，所以执行了一次 `System.out.println(System.getProperty("file.encoding"))`发现打印出来的是`GBK`。那问题也就明了：
在 XML 转换成字节序列时，使用`GBK`编码转换，而在从字节序列转换成字符串时按照 UTF-8 编码去解析的，所以报`Invalid byte 2 of 2-byte UTF-8 sequence`，也就不奇怪了。  
那么为什么是`Invalid byte 2 of 2-byte UTF-8 sequence`而不是`Invalid byte 2 of 3-byte UTF-8 sequence`或者是`Invalid byte 3 of 3-byte UTF-8 sequence`呢？简单来说 GBK 编码的字节序列，在 UTF-8 编码下解析时，UTF-8 识别到根据某一个字节识别到当前连续的两个字节为一个字符，所以再解析第二个字节，但是发现第二个字节不符合 UTF-8 二字节字符的第二字节的编码规则，所以就报了`Invalid byte 2 of 2-byte UTF-8 sequence`。

UTF-8 的编码算法如下：  

0. 0x00-0x7F 这个范围的 Unicode 字符使用一个字节编码，其最高位为 0；
1. 所有的编码为多字节的的字符编码，非首字节的第一个位为 1，第二位为 0；
2. 0x080-0x7FF 这个范围的 Unicode 字符使用两个字节编码，第一个字节前两位为 1，第三位为0；
3. 0x0800-0xFFFF 这个范围的 Unicode 字符使用三个字节编码，第一个字节的前三位为 1，第四位为 0；
4. 0x010000-0x10FFFF 这个范围的 Unicode 字符使用四个字节编码，第一个字节的前四位为 1，第五位为 0。

对于二字节字符，第一个字节为`110x xxxx`，第二个字节为`10xx xxxx`，所以如果检查到某一个字节是`10xx xxxx`，那么就会去检查第二个字节头两位是否是`10`，如果不是就认为这不是一个有效的 UTF-8 字符。比如`中文`这个词中，`中`这个字符的 GBK 编码为`1101 0110 1101 0000`，通过 UTF-8 的编码规则解析时，读到第一个字节`1101 0110` 识别到它是一个二字节字符的第一个字节，那么第二个字节就应该是`10xx xxxx` 这样的，但是实际上第二个字节是`1101 0000`，这个时候就报了这个二字节的 UTF-8 字符的第二个字节无效，即`Invalid byte 2 of 2-byte UTF-8 sequence`。

## Reference
http://guimy.me/%E5%AD%97%E7%AC%A6%E9%9B%86%E4%B8%8E%E5%AD%97%E7%AC%A6%E7%BC%96%E7%A0%81/2017/04/09/charset_and_charencoding.html

