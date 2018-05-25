遇到一个编码的问题，在解析 TDDL 的规则文件时，报了
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
报`Invalid byte 2 of 2-byte UTF-8 sequence.`这个错误，很多文章的办法是将为 encoding 改为 GBK 编码，但是为什么呢？
以为遇到 JDK 的 bug 了。debug 进去排查，发现从 XML 文件解析出来的编码也是 UTF-8。那就奇怪了，那问题就出在将 XML 文本转换成字节数组的时候，和 xerces 解析字节数组的过程中。
再看看
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
这段代码中 `stringXmls[i].getBytes()` 将字符串转换为字节数组使用了默认的编码，看看`String.getBytes`，
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
