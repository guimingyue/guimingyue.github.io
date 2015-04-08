---
layout: post
title: MySQL协议:建立Socket连接
category: MySQL
---

##MySQL协议:建立Socket连接

MySQL协议用于客户端与服务端之间的通信，在Connector/C, Connector/J，MySQL Proxy，主从复制等都有对应实现。这篇文章将以最简单的客
户端视角来看客户端与服务端的通信协议中建立连接这一过程，下面的代码与说明只是针对capability中CLIENT_SECURE_CONNECTION为1的情况。

###MySql数据类型
客户端与服务端通信的过程中会用到各种类型的数据，MySQL协议定义了两种数据类型：整型和字符串类型。这两种数据类型分别有多种子类型，即固定长度和可变长度。在MySQL的通信协议中，规定了数据包中某些字段的类型，如果是固定长度，则会指定该字段的所占有的字节数，解析时只需按照对应的字节数读取，对于可变长度类型，根据数据包具体的长度进行解析。
####整型
整型分两种，分别是固定长度的整型（Fixed-Length Integer Types）和可变长度的整型（Length-Encoded Integer Type）。根据数据占用的比特数量不同，固定长度的整型又有6种，分别是：

* int<1> 占用1个字节
* int<2> 占用2个字节
* int<3> 占用3个字节
* int<4> 占用4个字节
* int<6> 占用6个字节
* int<8> 占用8个字节

可变长度的整型，具体可以参考[http://dev.mysql.com/doc/internals/en/integer.html][2]

####字符串类型
字符串类型用于文本信息，主要分为四种：

* FixedLengthString，这种类型的字符串是定长的，如`ERR_Packet`中的`sql-state`字符串的长度为5，在MySQL协议中，定长字符串字段根据其上都读取相应的
字节数
* NulTerminatedString，这种类型的字符串长度不固定，连续并且以'\0'结束，类似于C语言中的字符串。
* VariableLengthString，长度不固定，其长度在运行时确定或者由另一个字段指定。
* LengthEncodedString，由长度和字符串两个部分组成，在MySQL协议中，根据长度字段读取相应的字节数。
* RestOfPacketString，该字符串是报文的当前索引到报文末尾部分的整体。

具体字符串的类型可以参考[http://dev.mysql.com/doc/internals/en/string.html][3]

###报文
客户端与服务端的通信以报文进行，并且一个报文最大为16MB，如果发送的数据大于16MB，那么就将该数据划分成多个报文传输。报文的格式如下：

    | 有效载荷长度 | 报文序号 | 有效载荷 |

* 其中有效载荷为长度占3个字节，它表示该报文中有效载荷的字节数
* 报文序号占一个字节，依次递增，从0开始
* 有效载荷即所发送的具体数据

对于客户端发送给服务器的绝大部分命令，服务端返回如下三种响应报文之一：

1. OK_Packet,
2. ERR_Packet
3. EOF_Packet

关于这三种报文的格式可以参考MySQL官方文档[http://dev.mysql.com/doc/internals/en/generic-response-packets.html][1]

###连接的生命周期

MySQL协议中一个连接分为两个阶段：建立连接阶段和命令阶段。其中建立连接即客户端建立与服务端的Socket连接，
命令阶段就是客户端通过已经建立的连接发送待执行的命令到服务端执行，然后返回结果。本文仅仅讨论连接建立这个过程。

一个建立至少经过了四个步骤：

1. 客户端发起Socket连接请求；
2. 服务端接受连接建立请求，并且返回一个初始化的报文；
3. 客户端利用初始化的报文，发送客户端的认证信息给服务端；
4. 服务端认证客户端，将认证结果发送给客户端。

认证成功之后，连接就建立完成，客户端就可以给服务端发送命令了。下面就以客户端的视角，通过Java代码介绍每一个步骤的执行流程。

####客户端发起连接请求

首先客户端通过Socket，向请求建立连接，代码如下：

```java

SocketChannel channel = SocketChannel.open();
channel.socket().setKeepAlive(true);
channel.socket().setReuseAddress(true);
channel.socket().setSoTimeout(30 * 1000);
channel.socket().setSendBufferSize(16 * 1024);
channel.socket().setReceiveBufferSize(16 * 1024);
channel.socket().setTcpNoDelay(true);
channel.connect(new InetSocketAddress("127.0.0.1", 3306));
```
常规的Java NIO网络Socket客户端请求建立连接的一段代码，客户端首先创建SocketChannel对象，再设置TCP相关参数，然后请求与
服务端建立连接。

####服务端发送握手报文

服务端收到客户端建立连接的请求之后，就接受该请求，建立两端的TPC连接，这样服务端就可以和客户但通信了。在MySQL协议中，客户端与
服务端建立TCP连接后，服务端首先向客户端发送一个初始化握手报文，该报文的有效载荷格式如下：

```java
1              [0a] protocol version
string[NUL]    server version
4              connection id
string[8]      auth-plugin-data-part-1
1              [00] filler
2              capability flags (lower 2 bytes)
  if more data in the packet:
1              character set
2              status flags
2              capability flags (upper 2 bytes)
  if capabilities & CLIENT_PLUGIN_AUTH {
1              length of auth-plugin-data
  } else {
1              [00]
  }
string[10]     reserved (all [00])
  if capabilities & CLIENT_SECURE_CONNECTION {
string[$len]   auth-plugin-data-part-2 ($len=MAX(13, length of auth-plugin-data - 8))
  if capabilities & CLIENT_PLUGIN_AUTH {
string[NUL]    auth-plugin name
  }

```

首先是一个字节协议版本，然后一个变长字符串为服务端的MySQL版本，Capabiligy flags用于客户端和服务器之间协商它们支持那些特性，使用什么特性。

解析该报文的代码如下：

```java

byte[] bytes = readHeader(channel); //读取服务端发送的初始化握手报文的头部

int packetLen = getPacketLen(bytes);//报文有效载荷的长度
int sequence = bytes[3];//序号
byte[] body = readPacket(channel, packetLen);//有效载荷
HandshakeV10Packet packet = buildHandshakePacket(body);//解析初始化握手报文


//定义了报文的用到各个的字段
class HandshakeV10Packet {
    private byte protocolVersion;

    private String serverVersion;

    private int connectionId;

    private byte[] authPluginData;

    private int capability;

    private byte characterSet;

    private short statusFlags;

    private String authPluginName;
}

//解析初始化握手报文
public HandshakeV10Packet buildHandshakePacket(byte[] body) {
    HandshakeV10Packet packet = new HandshakeV10Packet();
    int index = 0;
    packet.setProtocolVersion(body[index]);//协议版本
    index++;
    String version = readNullEndStr(body, index);//读取服务端的MySQL版本，字符串类型，以\0结尾
    index += version.getBytes().length;
    packet.setServerVersion(version);

    int connectionId = readConnectionid(body, index);//connectionId
    packet.setConnectionId(connectionId);
    index += 4;

    byte[] authPluginData = readFixByte(body, index, 8);//auth-plugin-data-part-1
    index += 8;
    //一个字节的填充
    index += 1;

    short capabilityLowerByte = readFixByteShort(body, index);//capability的低2位
    index += 2;

    byte characterSet = body[index++];//char set，一个字节，MySQl支持多种字符集
    packet.setCharacterSet(characterSet);
    short statusFlag = readFixByteShort(body, index);//status flags
    index += 2;
    packet.setStatusFlags(statusFlag);

    int capability = readFixByteShort(body, index) << 16 | capabilityLowerByte;//capability高2位
    index += 2;
    packet.setCapability(capability);

    int authPluginDataLen = 0;

    if((capability & CLIENT_PLUGIN_AUTH) != 0) {//auth-plugin-data-part-2所占字节数
      int len = body[index];
      index++;
      authPluginDataLen = len-8;
    }

    index += 10;//保留10个字节空间

    if((capability & CLIENT_SECURE_CONNECTION) != 0) {
      byte[] authPluginData2 = readFixByte(body, index, authPluginDataLen);
      Charset charset = Charset.forName("ASCII");
      String s = charset.decode(ByteBuffer.wrap(authPluginData2, 0, authPluginData2.length)).toString();
      s = s.trim();//去掉最后一个字节
      authPluginData2 = s.getBytes();
      if(authPluginData2 != null && authPluginData2.length != 0) {
        byte[] newAuthPluginData = new byte[authPluginData.length + authPluginData2.length];
        System.arraycopy(authPluginData, 0, newAuthPluginData, 0, authPluginData.length);
        System.arraycopy(authPluginData2, 0, newAuthPluginData, authPluginData.length, authPluginData2.length);
        authPluginData = newAuthPluginData;
      }
      index += 13;
    }
    packet.setAuthPluginData(authPluginData);//随机字符串
    if((capability & CLIENT_PLUGIN_AUTH) != 0) {
      String authPluginName = readNullEndStr(body, index);
      index = index + authPluginName.getBytes().length;
      packet.setAuthPluginName(authPluginName);
    }
    return packet;
  }

  public static short readFixByteShort(byte[] body, int index) {
    short val = (short) ((body[index] & 0xff) | ((body[index+1] & 0xff) << 8));
    return val;
  }

  public static byte[] readFixByte(byte[] body, int index, int len) {
    int i = 0;
    byte[] buf = new byte[len];
    while(i < len) {
      buf[i] = body[index+i];
      i++;
    }
    return buf;
  }

  public static String readFixByteString(byte[] body, int index, int len) {
    return new String(readFixByte(body, index, len));
  }

  public static int readConnectionid(byte[] body, int index) {
    int connectionId = (body[index] & 0xff) | ((body[index+1] & 0xff) << 8) |
        ((body[index+2] & 0xff) << 16) | ((body[index+3] & 0xff) << 24);
    return connectionId;
  }

  public static String readNullEndStr(byte[] body, int index) {
    int start = index;
    while(body[index++] != 0);
    String version = null;
    if(start < index-1) {
      version = new String(body, start, index-start);
    }
    return version;
  }
```

客户端可以根据服务端的版本，capability等参数发送相应的响应参数，此外在当前的MySQL版本中，为了建立连接过程中的安全性，服务端在初始化
握手报文中会发送一个随机字符串，如果使用了`CLIENT_SECURE_CONNECTION`，这个字符串由两部分组成，`auth-plugin-data-part-1`和
`auth-plugin-data-part-2`。MySQL官方文档中这个字符串占用21个字节，但是`auth-plugin-data-part-1`已经有8个字节了，所以
`auth-plugin-data-part-2`占用13个字节，`auth-plugin-data-part-2`最后一个字节为0x00，并且在客户端根据该字符串进行计算时，必须将
最后一个字节排除，否则服务端会认证失败。在MySQL的JDBC驱动的实现中，遇到最后这一个字节，直接终止循环。

####客户端响应握手报文
客户端收到服务端的初始化握手报文后，首先根据握手报文中的随机字符串和登陆MySQl的密码，计算出一个另一个随机字符串，MySQL对该字符串的
生成规则描述是：

```java

SHA1( password ) XOR SHA1( "20-bytes random data from server" <concat> SHA1(SHA1(password)))

```
首先对密码进行两次SHA-1运算，然后与服务端的20个字节的随机数据连接，再与密码的一次SHA-1结果作异或运算，客户端以这样得到的结果数据
作为密码部分进行认证，该运算的实现代码入下：

```java

public byte[] encrypterPassword(byte[] password, byte[] seed) {
    byte[] buf = null;
    try {
      MessageDigest sha1 = MessageDigest.getInstance("SHA-1");
      byte[] pass1 = sha1.digest(password);
      sha1.reset();
      byte[] pass2 = sha1.digest(pass1);
      sha1.reset();
      sha1.update(seed);
      byte[] pass3 = sha1.digest(pass2);
      for(int i = 0; i < pass3.length; i++) {
        pass3[i] = (byte) (pass3[i] ^ pass1[i]);
      }
      buf = pass3;
    } catch (NoSuchAlgorithmException e) {
      e.printStackTrace();
      buf = new byte[1];
      buf[0] = 0x00;
    }
    return buf;
  }

```
此外客户端还需要发送MySQl账户名，capability等参数，实现代码如下：

```java
public static byte[] buildHandshakeResponse41(HandshakeV10Packet packet) throws IOException {
    ByteArrayOutputStream out = new ByteArrayOutputStream();

    /**
     * CLIENT_LONG_PASSWORD CLIENT_LONG_FLAG CLIENT_PROTOCOL_41
     * CLIENT_INTERACTIVE CLIENT_SECURE_CONNECTION
     */
    int capability = 1 | 4 | 512 | 8192 | 32768;
    writeInt(capability, out);

    writeInt(1 << 24, out);

    writeByte((byte)33, out);//utf8_general_ci

    out.write(new byte[23]);//reserved 23 bytes

    out.write("root".getBytes());
    out.write(0x00);

    byte[] authResponse = encrypterPassword("123456".getBytes(), packet.getAuthPluginData());
    out.write(authResponse.length);
    out.write(authResponse);

    out.write(packet.getAuthPluginName().getBytes());
    out.write(0x00);

    return out.toByteArray();
}

```
将这个报文发送给服务端：

```java
byte[] responseBytes = buildHandshakeResponse41(packet);//创建发送报文的有效载荷

byte[] header = buildResponseHeader(responseBytes, sequence+1);//创建报文头部
channel.write(new ByteBuffer[]{ByteBuffer.wrap(header), ByteBuffer.wrap(responseBytes)});//发送

```
####服务都返回认证结果
服务端接收到客户端响应的报文后，进行认证，如果认证成功就返回OK_Packet，否则返回ERR_Packet，也有可能返回EOF\_Packet。在客户端根据服务端的随机
字符串计算应该发送的随机字符串的过程中，如果计算的结果不正确，则会返回MySQl的1045错误码。如果客户端发送的报文格式不正确，则会返回EOF_Packet，
如提示Bad Shakehand。

##Reference
* [http://dev.mysql.com/doc/internals/en/overview.html][4]
* [http://blog.csdn.net/siddontang/article/details/21161357][5]
* [http://cenalulu.github.io/mysql/myall-about-mysql-password/][6]
* [http://sebug.net/node/t-1847][7]
* [http://hutaow.com/blog/2013/11/06/mysql-protocol-analysis][8]


[1]: http://dev.mysql.com/doc/internals/en/generic-response-packets.html
[2]: http://dev.mysql.com/doc/internals/en/integer.html
[3]: http://dev.mysql.com/doc/internals/en/string.html
[4]: http://dev.mysql.com/doc/internals/en/overview.html
[5]: http://blog.csdn.net/siddontang/article/details/21161357
[6]: http://cenalulu.github.io/mysql/myall-about-mysql-password/
[7]: http://sebug.net/node/t-1847
[8]: http://hutaow.com/blog/2013/11/06/mysql-protocol-analysis
