---
layout: post
title: 以 Unicode 的视角看字符集与字符编码
category: 字符集与字符编码
---

## 概述

在平常的工作当中，每个同学都遇到过各种乱码问题，很多时候，解决乱码问题的解决方案是全部设置为 UTF-8 编码，但是为什么呢？字符集与字符编码有何关系？什么时候使用字符集？什么时候使用字符编码？这篇文章主要介绍了 Unicode 字符集和与之相关的几种字符编码，


## 字符集与字符编码

在计算机的世界中，任何数据都是以二进制的形式存在的，而人们在使用计算机的过程中看到的却是易于理解的字符（本文中不考虑图片，视频等数据）。所以在人们看到的字符与计算机处理的二进制之间，必然存在着某种转换规则。这种转换规则将人们从输入设备输入的字符转换为二进制进行处理， 将二进制转换成易于理解的字符输出到输出设备上。

### ASCII 字符集与编码

最原始的字符集与字符编码是大家所熟知的 ASCII（American Standard Code for Information Interchange） 码，它使用一个字节 7 位来表示一个字符，所以这个字符集有 128 个字符，其编码方式也非常简单将该字符集的各个字符与一字节的二进制编码一一对应起来，这样就形成了著名的 ASCII码表<sup>[1]</sup>。

### Unicode 与 UTF-16

为了便于理解，简单的讲字符集就是人们使用到的字符的集合的总称，而字符编码则是字符与二进制之间的转换规则。所以字符集其实是一个范围，而字符编码则是一种具体的实现。最常见的 Unicode 字符集，也称为通用字符集（Universal Multiple-Octet Coded Character Set，UCS），它包括了当前已知语言的所有字符<sup>[2]</sup>。那么对于 Unicode 字符集，它对应的编码规则是怎么样的呢？Unicode 的编码有 UCS-2 和 UCS-4。UCS-2 指使用两个字节（16 位）来存储一个 Unicode 字符，所以 UCS-2 总共有 65536 个字符，同样的，UCS-4 就是指使用四个字节来存储一个 Unicode 字符。Unicode 最初的版本是使用 UCS-2 来编码每一个字符，但是随着 Unicode 标准的发展，新加入的字符不断增多，很多 emoji 字符也被加入到 Unicode 字符中，所以 UCS-2 不能满足 Unicode 字符的需要，于是，又出现了一个 UTF-16，它是一种变长编码方案，使用2个字节或者4个字节来表示一个字符。那么 UTF-16 如何将每个字符编码成二进制呢？很自然的想法是将 Unicode 中的每个字符先进行一次全排序，然后每个字符分配一个序号，使用每个字符的序号值作为其对应的二进制编码，这样做是正确的，因为任何编码的思路都是源于这种思路，但是仔细思考一下， Unicode 字符集的字符数量巨大，使用 16bit 只能表示 65536 个字符，那么两个字节肯定不能容纳所有的字符，所以应该使用四个字节来表示一个字符，但是这样就造成了空间的浪费，因为常用字符不到 65536 个，使用两个 16bit 就够了。

Unicode 的编码空间（0x0000-0x10FFFF）分为 17 个平面，每个平面最多可以表示 65536 个字符，第一个平面中的字符是常用字符，称为基本多语言平面（Basic Multilingual Plane, BMP），其他平面称为辅助平面。另外在对字符进行编码时，每个字符对应的编码位置称为码位（code point）。所以 UTF-16 使用两个字节来表示基本多语言平面中的字符，范围是 0x0000-0xFFFF，需要注意的是这个范围中 0xD800 到 0xDFFF 之间的码位不映射到任何 Unicode 字符，它们被用来对辅助平面的字符的码位进行编码。

UTF-16 是如何对 Unicode 字符进行编码呢？为了解释清楚 UTF-16 的编码方案，首先将 Unicode 的编码空间（0x0000-0x10FFFF）划分成四段，而这四的前三段组成一个区间，剩下的一段组成一个区间，如图 1 所示。

![图1 UTF-16 编码](/images/posts/UTF-16.png)
__*图1 UTF-16 编码空间*__

图 1 中，编码空间 0x0000-0x10FFFF 被划分编号为1，2，3，4 的四段，分别是 0x0000-0xD7FF，0xD800-0xDFFF，0xE000-0xFFFF，0x10000-0x10FFFF。其中，段 1，2，3 组成了区间一，对应基本多语言平面，段 4 组成了区间二，对应辅助平面。在区间一中，段 1 和段 3 这个范围内的每个值都是基本多语言平面内的编码码位（coide point），而段 2 范围内的值则用于对辅助平面的字符进行编码，这样在编码与解码的时候，就可以很方便的识别出哪些码位是基本多语言平面字符，哪些是辅助平面字符。

UTF-16 的编码算法如下<sup>[4]</sup>：
1. 0x0000-0xD7FF（段 1），0xE000-0xFFFF（段 3） 这两个范围内的值等价于对应的码位（code point）；
2. 0x10000-0x10FFFF（段 4） 这个范围内的值先减去 0x10000 得到的值在范围 0x0000-0xFFFFF 内，这个值的高 12 位全是 0，不需要做运算，将低 20 位划分成两部分，分别为高 10 位与低 10 位，这两个部分的值都在 0x000-0x3FF 这个范围内：
    1. 将 0xD800-0xDFFF（段 2） 再划分为两个范围，分别是 0xD800-0xDBFF，0xDC00-0xDFFF；
    2. 将高 10 位的值加上 0xD800 得到一个值，这个值的范围是 0xD800-0xDBFF，称为高代理位（high surrogate）；
    3. 将低 10 位的值加上 0xDC00 得到一个值，这个值的范围是 0xDC00-0xDFFF，称为低代理位（low surrogate）。

可以看到高代理位与低代理位的值范围与段 2 对应。所以对于 UTF-16 编码，可以根据每个字节的值所在的范围判断它是基本多语言平面内的编码码位还是辅助平面的高（低）代理位。


### Java 中的字符串

在 Java 世界中，基本类型`char`占 2 个字节，即 16 位，所以`char`类型只能表示 Unicode 字符集的基本多语言平面中的字符，对于辅助平面需要 4 个字节来表示<sup>[3]</sup>。Java 的`String`和`Character`类提供了一系列方法支持，比如下面一段代码。

```
@Test
public void testStringValueOf() {
    int codePoint = 0x10151;//辅助平面中的古希腊数字

    String s;
    if(Character.charCount(codePoint) ==  1) {//判断这个 codePoint 是否是基本多语言平面中的字符
        s = String.valueOf((char) codePoint);
    } else {
        s = new String(Character.toChars(codePoint));//对于辅助平面的字符通过 Character.toChars(codePoint) 转换为 char 数组
    }

    Assert.assertEquals(s.codePointCount(0, 1), 1);//字符串 s 的第 0 和 1 个字符组成一个 codePoint，也就是说其实只有一个字符
    Assert.assertEquals(Integer.toHexString(s.codePointBefore(2)), "10151");
    Assert.assertEquals(Integer.toHexString(s.codePointAt(0)), "10151");
}
```

上面这段代码中，对于 值为`0x10151`的 code point，`Character.toChars(codePoint)`返回的字符数组长度为 2，说明这个辅助平面字符在 Java 中由两个字节表示。除了上面的这段代码中使用到的几个方法，还有其他的一些与 code point 方法可供使用<sup>5</sup>

### UTF-8


UTF-16 一般用于内存中处理数据，因为常用的字符都在基本多语言平面中，在内存中基本上可以认为字符都是定长的，便于随机访问字符。但是在传输和存储过程中，如果待处理的字符仅仅是 ASCII 字符，那么继续使用 UTF-16 编码无疑消耗太大，所以针对传输和存储一般使用更省空间“的 UTF-8 字符编码。为什么 UTF-8 编码更省空间呢？因为它是一种比 UTF-16 更灵活的变长编码方案。比如对于 ASCII 码，UTF-8 仅使用一字节。

UTF-8 的编码方式如图 2 所示。

![图2 UTF-8](/images/posts/UTF-8.png)
__*UTF-8编码空间*__


图 2 同样将 Unicode 编码空间 0x0000-0x10FFFF 被划分编号为1，2，3，4 的四段，这四段中的字符占用的空间分别是 1 个字节，2 个字节，3 个字节，4 个字节。所以对于 ASCII 码来说，相对于 UTF-16 编码，UTF-8 编码可以省一半的空间，但是对于中文，韩文这些字符（在段 3 的范围中），相对于 UTF-16 却占用更多的空间。那么为何要使用 UTF-8 编码传输存储数据呢？UTF-8 编码无缝兼容 ASCII 编码，UTF-8 的编码算法<sup>[6]</<sup>如下：

1. 0x00-0x7F 这个范围的 Unicode 字符使用一个字节编码，其最高位为0；
2. 所有的编码为多字节的的字符编码，非首字节的第一个位为 1，第二位为0；
3. 0x080-0x7FF 这个范围的 Unicode 字符使用两个字节编码，第一个字节前两位为 1，第三位为0；
4. 0x0800-0xFFFF 这个范围的 Unicode 字符使用三个字节编码，第一个字节的前三位为 1，第四位为 0；
5. 0x010000-0x10FFFF 这个范围的 Unicode 字符使用四个字节编码，第一个字节的前四位为 1，第五位为 0。

从这个算法可以看出，解析 UTF-8 编码的数据，不需要对数据进行复杂的计算（相比于 UTF-16），或许这也是 UTF-8 编码常用于传输和存储的原因之一吧！

## 总结

上文首先从字符集与字符编码的角度解释了字符集与字符编码的关系，并且详细介绍了 UTF-16 的编码方式，然后以一个 Java 程序示例介绍了 UTF-16 的一种使用方式，最后介绍了 UTF-8 编码。UTF-8，UTF-16以及本文未讨论的 UTF-32 这几种编码只是 Unicode 字符集的不同编码方式。在中文世界中，其他常见的字符编码有 GBK，GB2312，GB18030，这几种字符集都不属于 Unicode，所以在本文不做讨论。

EOF

## Reference

[1] https://zh.wikipedia.org/wiki/ASCII

[2] https://zh.wikipedia.org/wiki/%E9%80%9A%E7%94%A8%E5%AD%97%E7%AC%A6%E9%9B%86

[3] https://docs.oracle.com/javase/tutorial/i18n/text/unicode.html

[4] https://zh.wikipedia.org/wiki/UTF-16 

[5] https://www.ibm.com/developerworks/library/j-unicode/

[6] https://zh.wikipedia.org/wiki/UTF-8

