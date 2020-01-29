---
layout: post
title: 深入分析 Linkedin 开源键值数据库 PalDB
category: PalDB
---

## PalDB 简介
PalDB 是 LinkedIn 开源的一个嵌入式的键值数据库，使用 Java 语言编写，GitHub 地址是 [https://github.com/linkedin/PalDB](https://github.com/linkedin/PalDB)，但是与 leveldb 等键值数据库不同的是，PalDB 只支持写入一次数据，并且写入过程中仅仅只能写入不支持读数据操作。所以其使用场景就是数据写入一次后，后续只需要读取。LinkedIn 使用其来存储[side data](https://github.com/linkedin/PalDB#what-is-it-suitable-for)。

## PalDB 的使用
使用 PalDB 时，会将其嵌入到一个应用中，该应用汇调用 PalDB 的接口将键值对写入到文件中。一个 PalDB 数据文件，仅支持写入一次，写入后的数据供嵌入 PalDB 的应用来读取，所以 PalDB 的写入和读取的示例如下。

* 写入
```java 
StoreWriter writer = PalDB.createWriter(new File("store.paldb"));
writer.put("foo", "bar");
writer.put(1213, new int[] {1, 2, 3});
writer.close();
```
* 读取
```java
StoreReader reader = PalDB.createReader(new File("store.paldb"));
String val1 = reader.get("foo");
int[] val2 = reader.get(1213);
reader.close();
```
可以看到 API 比较简单，无非就是打开一个文件，获取写入或者读取对象，然后就是做对应的操作。由于操作数据的写入接口（`StoreWriter`）和读取接口（`StoreReader`）是两个不同的接口，所以也无法实现在写入的时候做读取操作。需要注意的是创建写入对象时，如果传入的参数是文件对象，那么已经存在的文件是会被覆盖的，而且 PalDB 在写入文件时会先将元数据写在文件头部，然后是索引和数据。所以也无法实现对一个 PalDB 的数据文件做附加数据操作。
如果按照 LinkedIn 介绍的使用场景分析 PalDB 的作用，更合适的不是写入本地文件，而是写入像 HDFS 这样的分布式文件系统，然后供后续的机器学习，数据分析等应用使用。

## 存储文件结构
PalDB 在写入数据时（调用`put`接口）仅仅是将数据和索引写入到临时文件和内存中，只有调用`com.linkedin.paldb.api.StoreWriter#close`接口时才将数据写入数据文件持久化，所以通过查看`close`接口的代码很容易知道 PalDB 的文件存储结构，详细的代码可以参考 PalDB 的代码 `com.linkedin.paldb.impl.StorageWriter#close`，这个接口主要做了以下几件事。

0. 将当前 PalDB 的最新版本，当前时间，键的总数和长度等信息写入到一个临时元数据文件中。
1. 为键的字节数相同的键创建索引数据的临时索引文件。
2. 合并上两步创建的临时元数据文件和临时索引文件，以及数据写入过程中生成的临时数据文件，合并后的文件就是本次写入流程创建 PalDB 的文件。

根据这个流程，PalDB 的数据文件格式总结如下。
```shell
// 元数据
|PalDB 最新版本（String）|写文件时间戳（long）|不同的键（key）总数(int)|不同长度的键（key）的个数（int）|所有键的字节数组长度的最大值（int）|
---------------------------------------------------------------------------------------------------------------------------
[|键长（字节数）|键的数量|slot 数量|slot size|该长度的键在数据文件中的索引偏移|该长度的数据在数据文件中的数据偏移|,...]
---------------------------------------------------------------------------------------------------------------------------
[复合类型的自定义序列化配置]
---------------------------------------------------------------------------------------------------------------------------
|索引在数据文件中的偏移|数据在数据文件中的偏移|
---------------------------------------------------------------------------------------------------------------------------
// 索引数据
[|键|值的偏移地址|，|键|值的偏移地址|，...]
---------------------------------------------------------------------------------------------------------------------------
// 值数据
[|值长度|值|，|值长度|值|]
```
PalDB 内存和文件中的索引数据以键（key）的字节数为单位进行组织，也就是如果键（key）的字节数相同，那么他们就是一类，这个 key 就会被放入与其长度一致的索引中，而 PalDB 对于这块的实现则更为简单，在代码中使用数组来存储部分索引信息数据，而数组元素的下标就是 key 的字节数量。如果大量的 key 都是比较短的，但是仅有小部分 key 的特别长，那么可能会有大部分的数组元素为 null，这样导致的结果是浪费了一部分内存空间用来分配数组。
对于文件中的索引数据，在确定写入到的文件位点时，会先根据键的字节数组计算出一个哈希值，然后再利用哈希值与探测次数（probe）的和对 slots 取模，得到其所在的 slot，文件的位点就是`slot * slotSize`，这样就可以往该位点写入数据了。如果该位点已经写入了数据，就说明遇到冲突了，那么就要继续探测来解决冲突，即 `probe++`。相关代码如下。

```java
// Hash  
long hash = (long) hashUtils.hash(keyBuffer);  
  
boolean collision = false;  
for (int probe = 0; probe < count; probe++) {  
  int slot = (int) ((hash + probe) % slots);  
  byteBuffer.position(slot * slotSize);  
  byteBuffer.get(slotBuffer);  
  
 long found = LongPacker.unpackLong(slotBuffer, keyLength);  
 if (found == 0) {  
    // The spot is empty use it  
  byteBuffer.position(slot * slotSize);  
  byteBuffer.put(keyBuffer);  
 int pos = LongPacker.packLong(offsetBuffer, offset);  
  byteBuffer.put(offsetBuffer, 0, pos);  
 break;  } else {  
    collision = true;  
  // Check for duplicates  
  if (Arrays.equals(keyBuffer, Arrays.copyOf(slotBuffer, keyLength))) {  
      throw new RuntimeException(  
              String.format("A duplicate key has been found for for key bytes %s", Arrays.toString(keyBuffer)));  
  }  
  }  
}
```

由于 PalDB 不允许有重复的键，所以不同的键总数就是整个 PalDB 的键的个数。另外需要注意的是，PalDB 在写入数据过程中创建的所有临时文件在进程退出时，会自动删除（`java.io.File#deleteOnExit`）。如果进程由于某些原因，在调用 `close` 接口前退出了，那之前写入的数据就都丢了，所以这大概也是写数据过程中无需 flush 数据的原因吧，因为根本就不需要考虑未 flush 到磁盘的数据，反正最终生成的数据文件在关闭数据库时才真正创建。

## 写数据的内存结构
在调用 PalDB 的`put`接口时，最终会走到`com.linkedin.paldb.impl.StorageWriter#put`这个方法，`StorageWriter`是 PalDB 内部写数据的具体实现类，其`put`方法的两个参数分别是键值对中键和值的字节数组。在该方法中，首先会获取索引文件的输出流（没有就创建），再写入键（key）的字节数组以及值（value）在数据文件中的偏移量，然后获取数据文件的输出流（没有就创建），最后写入值的长度和完整的字节。
整个写入过程涉及`com.linkedin.paldb.impl.StorageWriter`中定义变量如下：

* indexStreams：索引文件的输出流数组。
* indexFiles：索引文件句柄数组。
* keyCounts：各种键（key）长度计数数组。
* maxOffsetLengths：各种键（key）长度对应值（value）的最大偏移量数组，数组中每一个元素都是某个键值字节数对应的所有值的最大长度的编码。
* lastValues：各种键（key）长度的最后一个写入值字节数组
* lastValuesLength：各种键（key）长度对应的值（value）的最后写入的值写入数据文件的字节数数组（一个值写入数据文件有值的长度和值）
* dataLengths： 各种键（key）长度对应的所有值的长度数组（一个值写入数据文件有值的长度和值）。

这些成员变量均是数组类型，数组元素的下标就是键的字节数，比如字符 `a` 的字节数为 1，那么其以上的数组成员变量的位置就是第 1 个元素。需要说明的是，每一个值写入数据文件的数据有两部分，第一部分是值的长度，第二部分是指，所以在`dataLengths`和`lastValuesLength`数组中会计算加上这两部分的长度。


### 临时索引与数据文件索引的区别
在最终将数据从临时文件中写入到真正的数据文件中时，索引文件会进行重新组织，重新组织后的索引文件才会合并到最终的数据文件中，也就是数据文件中是包含索引数据的，那么这两种索引文件有什么区别呢？

* 临时索引文件
该文件中仅写入了键（key）的字节数组和值（value）在值索引文件中的偏移量，PalDB 按照键（key）的字节长度进行分组，每种字节长度都会对应一个临时索引文件。
* 数据文件索引
最终的数据文件中有一部分是索引数据，这部分索引数据包含的也是以 key 的字节数组和值的偏移，只是这部分索引数据是整个数据文件的一部分，取数据时，需要根据键（key）的长度去文件中找其对应的索引数据。

## 读数据的内存结构
读数据主要流程最后会走到`com.linkedin.paldb.impl.StorageReader`，在其初始化时，会读取制定 PalDB 的数据文件，读取其中的元数据，并初始化索引数据。
### StorageReader 的初始化
在 StorageReader 的构造方法中，会首先读取元数据，也就是根据数据文件的格式读取元数据，比如版本，时间戳，键的数量等基本信息，每一种长度的键的信息，自定义的序列化类型信息以及索引和数据在文件中的偏移量。
对于索引数据，会对这块文件数据通过 mmap 映射一个 `MappedByteBuffer` 对象，而对于数据部分，如果配置了 `mmap.data.enabled` 参数，会根据定义的 segmentSize 大小，可能会映射出一个 `MappedByteBuffer` 数组。

### 读取数据
读取数据时，会先计算出 key 对应的哈希值，然后根据 hash 值在索引文件部分进行探测，过程与写入索引数据相同，如果找到了对应的 key，就继续找其对应的值。读取数据的过程比较简单，就不继续分析了。

## 总结
PalDB 整体比较简单，功能也相对简单，仅仅是写入后读取数据，没有复杂的数据结构，在源码中用得最多的数据结构就是数组。PalDB 对于写入无事务保障，写入时，将数据写入临时文件中，最后关闭操作时才会将元数据，索引数据和键值数据一起写入文件中。
PalDB 在 GitHub 上的代码上次提交 commit 已经是 4 年以前了，所以应该不会继续维护了，不过通过这个项目可以看看一个最简单的 Key-Value 数据库是如何实现的。

## Reference
[Open-sourcing PalDB, a lightweight companion for storing side data](https://engineering.linkedin.com/blog/2015/10/open-sourcing-paldb--a-lightweight-companion-for-storing-side-da)  
[PalDB 介绍](https://www.jianshu.com/p/9de9a2ea482c)
