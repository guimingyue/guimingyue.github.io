---
layout: post
title: Invalid byte 2 of 2-byte UTF-8 sequence
category: Java
---
前几天在使用 Spring 解析一个 XML bean 文件时遇到了编码的问题，报错异常栈如下图所示。
```java
Exception in thread "main" org.springframework.beans.factory.BeanDefinitionStoreException: IOException parsing XML document from resource loaded from byte array; nested exception is com.sun.org.apache.xerces.internal.impl.io.MalformedByteSequenceException: Invalid byte 2 of 2-byte UTF-8 sequence.
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.doLoadBeanDefinitions(XmlBeanDefinitionReader.java:416)
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:342)
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:310)
	at org.springframework.beans.factory.support.AbstractBeanDefinitionReader.loadBeanDefinitions(AbstractBeanDefinitionReader.java:143)
	at org.springframework.context.support.AbstractXmlApplicationContext.loadBeanDefinitions(AbstractXmlApplicationContext.java:109)
	at org.springframework.context.support.AbstractXmlApplicationContext.loadBeanDefinitions(AbstractXmlApplicationContext.java:80)
	at org.springframework.context.support.AbstractRefreshableApplicationContext.refreshBeanFactory(AbstractRefreshableApplicationContext.java:123)
	at org.springframework.context.support.AbstractApplicationContext.obtainFreshBeanFactory(AbstractApplicationContext.java:422)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:352)
	at com.taobao.tddl.rule.utils.StringXmlApplicationContext.<init>(StringXmlApplicationContext.java:41)
	at com.taobao.tddl.rule.utils.StringXmlApplicationContext.<init>(StringXmlApplicationContext.java:27)
	at me.guimy.java8.Functions.main(Functions.java:563)
Caused by: com.sun.org.apache.xerces.internal.impl.io.MalformedByteSequenceException: Invalid byte 2 of 2-byte UTF-8 sequence.
	at com.sun.org.apache.xerces.internal.impl.io.UTF8Reader.invalidByte(UTF8Reader.java:701)
	at com.sun.org.apache.xerces.internal.impl.io.UTF8Reader.read(UTF8Reader.java:372)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityScanner.load(XMLEntityScanner.java:1895)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityScanner.scanData(XMLEntityScanner.java:1375)
	at com.sun.org.apache.xerces.internal.impl.XMLDocumentFragmentScannerImpl.scanCDATASection(XMLDocumentFragmentScannerImpl.java:1654)
	at com.sun.org.apache.xerces.internal.impl.XMLDocumentFragmentScannerImpl$FragmentContentDriver.next(XMLDocumentFragmentScannerImpl.java:3014)
	at com.sun.org.apache.xerces.internal.impl.XMLDocumentScannerImpl.next(XMLDocumentScannerImpl.java:602)
	at com.sun.org.apache.xerces.internal.impl.XMLDocumentFragmentScannerImpl.scanDocument(XMLDocumentFragmentScannerImpl.java:505)
	at com.sun.org.apache.xerces.internal.parsers.XML11Configuration.parse(XML11Configuration.java:841)
	at com.sun.org.apache.xerces.internal.parsers.XML11Configuration.parse(XML11Configuration.java:770)
	at com.sun.org.apache.xerces.internal.parsers.XMLParser.parse(XMLParser.java:141)
	at com.sun.org.apache.xerces.internal.parsers.DOMParser.parse(DOMParser.java:243)
	at com.sun.org.apache.xerces.internal.jaxp.DocumentBuilderImpl.parse(DocumentBuilderImpl.java:339)
	at org.springframework.beans.factory.xml.DefaultDocumentLoader.loadDocument(DefaultDocumentLoader.java:75)
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.doLoadBeanDefinitions(XmlBeanDefinitionReader.java:396)
	... 11 more

```
关键的报错信息时`Invalid byte 2 of 2-byte UTF-8 sequence.`这个错误，很多文章的办法是将为 encoding 改为 GBK 编码，但是为什么呢？那么我们先来简化代码，关键代码如下。
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
ApplicationContext applicationContext = new StringXmlApplicationContext(data, Functions.class.getClassLoader());
```
这段代码中加载一个 xml bean 文件，其中有一个值是中文，很显然在解析到中文的时候就报了`Invalid byte 2 of 2-byte UTF-8 sequence.`错。可以看到已经定义了 `encoding="UTF-8"`，所以理论上解析在解析的时候不应该无法解析中文的错误。难道是遇到 JDK 的 bug 了？从异常栈可以看到 Spring 在解析 XML 调用了 JDK 中`com.sun.org.apache.xerces.internal.jaxp`包下的类来解析，所以针对这个疑问，可以先 debug 到 具体的类中排查使用什么编码在解析 XML，其实也可以从异常栈中看到是使用 UTF-8 的编码在解析，但是通过 debug 可以了解到，这个程序执行过程中解析 XML 是解析的文本对应的字节序列。
既然知道了解析 XML 文本对应的字节序列是使用的 UTF-8 编码，并且文本中也指定了 UTF-8 编码，那么怀疑的方向就是字节序列是怎么生成的，是否是 UTF-8 编码对应的字节序列？所以现在需要确定的如何将 XML 文本转换成字节序列的。可以看看 Spring 的 `StringXmlApplicationContext` 构造方法。
```java
public StringXmlApplicationContext(String[] stringXmls, ApplicationContext parent, ClassLoader cl) {
        super(parent);
        this.cl = cl;
        this.configResources = new Resource[stringXmls.length];

        for(int i = 0; i < stringXmls.length; ++i) {
            this.configResources[i] = new ByteArrayResource(stringXmls[i].getBytes());
        }

        this.refresh();
    }
```
这段代码中 `stringXmls[i].getBytes()` 将字符串转换为字节数组使用了默认的编码，再看`String.getBytes`方法的代码，
```java
public byte[] getBytes() {
        return StringCoding.encode(value, 0, value.length);
    }
static byte[] encode(char[] ca, int off, int len) {
        String csn = Charset.defaultCharset().name();
        try {
            // use charset name encode() variant which provides caching.
            return encode(csn, ca, off, len);
        } catch (UnsupportedEncodingException x) {
            warnUnsupportedCharset(csn);
        }
        try {
            return encode("ISO-8859-1", ca, off, len);
        } catch (UnsupportedEncodingException x) {
            // If this code is hit during VM initialization, MessageUtils is
            // the only way we will be able to get any kind of error message.
            MessageUtils.err("ISO-8859-1 charset not available: "
                             + x.toString());
            // If we can not find ISO-8859-1 (a required encoding) then things
            // are seriously wrong with the installation.
            System.exit(1);
            return null;
        }
    }
```
再打印一下 Syste.getProperty("file.encoding")，原来是 GBK，那问题也就明了了：
在 XML 转换成字节数组时，使用 GBK 转换，而在从字节转换成字符串时按照 UTF-8 编码去解析的，所以报 Invalid byte 2 of 2-byte UTF-8 sequence，也就不奇怪了。

再解释 UTF-8 的中文编码和 GBK 的中文编码的区别，解释清楚问什么会报Invalid byte 2 of 2-byte UTF-8 sequence



![image](http://git.cn-hangzhou.oss-cdn.aliyun-inc.com/uploads/mingyue.gmy/Test/8bcc886f7dc373624740510120461cca/image.png)


