---
title: 深入理解Protobuf3协议原理
date: 2019-11-30 11:59:08
tags:
---

## Protobuf是什么

全称为Protocol Buffers，Google推出的序列化框架，用于将自定义数据结构序列化成字节流，和将字节流反序列化为数据结构，该框架不依赖开发语言，也不依赖运行平台，扩展性好，目前支持的语言比较多，包括Java，C++，Python，Ruby等。

## 使用Protobuf

在这里使用Windows和Java进行实例演示：

1. 先去[https://github.com/protocolbuffers/protobuf/releases](https://github.com/protocolbuffers/protobuf/releases "https://github.com/protocolbuffers/protobuf/releases")把protoc-3.11.0-win64.zip下载下来，然后解压，找到bin目录，cmd进去bin目录。

2. 去[https://repo1.maven.org/maven2/com/google/protobuf/protobuf-java/](https://repo1.maven.org/maven2/com/google/protobuf/protobuf-java/ "https://repo1.maven.org/maven2/com/google/protobuf/protobuf-java/")，找到最新版本，把jar下载下来。

3. 新建Animal.proto文件，编写如下代码：

		syntax="proto3";
		// option java_outer_classname = "ProtoBufAnimal"
		message Animal {
			int32 age = 1;
			string name = 2;
		}

	这里定义了一个消息类型Animal，它有两个字段，字段的定义包括类型，名称和字段号，这个字段号必须是唯一的数字，数字越大，占用的字节数也就越多，如果是1~15，那么它就只占用一个字节，因此频繁使用的字段的字段号应该尽量取小一点。

4. bin目录下打开cmd，执行命令编译Animal.proto文件：

		protoc Animal.proto --java_out .

	编译后生成了AnimalOuterClass类，可以把这个AnimalOuterClass类理解为辅助类，为了让我们能够很方便地操作proto定义的消息类型，它提供API让我们可以设置或者读取数据，也可以序列化和反序列化。

5. 打开Eclipse，新建Java项目，AnimalOuterClass类复制进来，第1步下载的jar也复制到libs目录下，然后add to build path。

6. 编写测试代码，分别测试序列化和反序列化：

		AnimalOuterClass.Animal.Builder builder = AnimalOuterClass.Animal.newBuilder();
		builder.setAge(12);
		builder.setName("haha");
		AnimalOuterClass.Animal animal = builder.build();
		// 序列化
		byte[] data = animal.toByteArray();
		// 十六进制形式打印
		for (byte b : data) {
			System.out.print(String.format("%02X", b & 0xFF));
		}
		System.out.println();
		// 反序列化
		AnimalOuterClass.Animal newAnimal = AnimalOuterClass.Animal.parseFrom(data);
		System.out.println("name:" + newAnimal.getName());
		System.out.println("age:" + newAnimal.getAge());

	输出结果为：

		080C120468616861
		name:haha
		age:12
	
	可以看到，序列化后得到的byte数组，只占用8个字节。

该demo项目源码：[https://github.com/dolpphins/ProtoBuf3Test](https://github.com/dolpphins/ProtoBuf3Test "https://github.com/dolpphins/ProtoBuf3Test")

## Protobuf3语法

### 指定协议类型

	syntax = "proto3";

如果不指定，就会默认使用版本proto2。Google在2001年创建了protobuf，用于内部使用，在2008年，Google把protobuf开源，定为protobuf2，到了2016年，又发布了protobuf3，目前还是有很多公司在使用protobuf2版本。

### 定义多个消息类型

	syntax="proto3";
	// option java_outer_classname = "ProtoBufAnimal"
	message Animal {
		int32 age = 1;
		string name = 2;
	}
	
	message Cat {
		int32 sex = 1;
	}

最终都会编译到AnimalOuterClass类中，作为它的一个内部类。

### 注释

注释跟大部分编程语言一样，行注释// 和 块注释/** */ 都可以。

### 字段类型

int32：可变长编码，也就是不是固定占用4个字节，一般用于保存正整数。

uint32：可变长编码。

sin32：可变长编码，存储负数时建议用这个。

fixed32：固定占用4个字节。

bool：布尔类型

string：字符串


## Protobuf原理

protobuf会对proto协议文件进行序列化，最终转换成二进制数据，在这个转换过程中，protobuf做了一些优化，使得转换出来的二进制数据尽可能的小，同时也具有安全，编解码快等特点。

### 消息类型编码

我们定义了一个Animal的消息类型，那protobuf是怎么把它编码成二进制的呢？其实它是把message转成一系列的key-value，key就是字段号，value就是字段值，大概这样子存：

	[tag1][value1][tag1][value2][tag3][value3]... 

解码时，会从左往右解析每一个key-value，假如遇到某个key-value无法解析了，那么就直接跳过，不会影响到其它key-value的解析，因此如果你加了新字段，生成字节流，然后用旧版本解析，这时它还是能够解析出旧版本的字段的，新字段只是被忽略而已，这就是protobuf的向后兼容。

另外注意到，实际上存储的是tag-value，而不是key-value，根据key转换成tag，也是有公式的：

	tag = (key << 3) | wire_type

这里的wire_type，其实就是数据类型对应的整数值，它们是预定义好的：

![avatar](/images/protobuf_wire_type.png)

而value也不是直接转成二进制就完事的，它会针对不同的数据类型做不过压缩编码，从而实现占用更少字节数的目的，实现方法下面会继续分析。

### 可变长编码类型

从上面的字段类型可以看到，像int32这些类型它并不是固定占用4个字节的，如果数值很小，那它可能只占一个字节，这就是可变长编码，很明显这样能够减少占用的空间。protobuf具体的实现方法，就是把每个字节的最高位做为标志位，1表示当前字节不是最末尾的字节，0表示当前字节是最末尾的字节，举个例子，protobuf将404表示为：

	10010100  00000011

它只占用两个字节，第一个字节的最高位为1，表示还需要读取下个字节，而第二个字节的最高位为0，表示不需要再读取下个字节了，因此这两个字节就是某个数值，那么要怎么计算究竟是哪个数组呢？protobuf的解码是这样的：

1. 去掉最高位，得到每个字节的低7位，也就是0010100 0000011。
2. 然后倒序，得到11 0010100。
3. 转成十进制，也就是404。

可以看到，可变长编码的核心，就是用字节的最高位做为标志位，用于确定是否需要读取下个字节，这样就可以节省一部分的字节占用了，比如404它就只占用了两个字节，不过，如果所要表示的数值很大，那通过这种方法，可能需要占用5个字节，比原本固定4个字节还多，不过大部分情况这种场景较少，如果真的有，也可以直接定义为固定字节的数据类型。

### 负数可变长编码类型

如果要表示的数是一个负数，采用上面所说的方式编码，就会占用很多字节，因为对于比较常用（数值比较大）的负数来说，转成补码后会有很多个1，也就是说占用的字节会比较多，这样如果还采用上面那种方式编码，就得不偿失了，因此Google又新增加了一种数据类型，叫sint，专门用来处理这些负数，其实现原理是采用zigzag编码，zigzag编码的映射函数为：

	ZigZag(n) = (n << 1) ^ (n << k)，k为31或者63

最终的效果就是把所有的整数映射为正整数，比如0->0, -1->1, 1->1, -2->3这样子，然后就可以用上面所说的编码方式进行编码了，解码时通过逆函数解析即可。关于ZigZag编码，可以参考[https://www.cnblogs.com/en-heng/p/5570609.html](https://www.cnblogs.com/en-heng/p/5570609.html "https://www.cnblogs.com/en-heng/p/5570609.html")这篇文章。

### 固定字节数类型

这种就很简单了，直接固定字节数就行，比如fixed32，它固定占用4个字节。而且可以看到，protobuf中float和double也是固定占用4个字节和8个字节，并没有实现压缩。

### string类型

按照固定的格式编码，格式为：

	value = length + content

其中length就是content占用的字节数，采用可变长编码，content就是string的具体内容。

## Protobuf实例解析

接下来根据上面所说的编码原理，解析最开始的实例项目所生成的二进制数据，实例项目proto代码为：

	syntax="proto3";
	// option java_outer_classname = "ProtoBufAnimal"
	message Animal {
		int32 age = 1;
		string name = 2;
	}

在代码里，age的值被设置为12，name的值被设置为haha，最终生成的二进制数据为：080C120468616861。

下面开始分析：

1. 首先message是通过tag-value存储的，所以这里其实就是两个tag-value。
2. 第一个tag-value对应的是age字段，其tag按照公式(key << 3) | wire_type计算，key为1，wire_type为0，最终结果为08，而value就是12采用可变长编码的结果，占用一个字节，最终结果为0C，因此第一个tag-value对应的就是前两个字节080C。
3. 第二个tag-value对应的是name字段，其tag同样按照公式计算，key为2，wire_type为2，最终结果为12，value按照string数据类型的编码格式可知，刚开始是长度，后面是内容，因此先看内容，haha的十六进制表示为68616861，占用4个字节，所有长度就是4，也就是04，整个value表示为0468616861。
4. 最后把两个tag-value拼接起来，就是080C120468616861，跟运行程序生成的一致。

可以看到，protobuf最终生成的二进制数据有个特点就是紧凑，几乎不包含冗余数据，因此数据也会比较小。


## Protobuf优缺点

### 优点

* 小：生成的字节流采用了各种压缩方式，相对xml和json这类文件更小。

* 快：编解码基本都是位运算，也没有复杂的嵌套关系，速度快。

* 安全：这里的安全，是指protobuf没有把字段名写入到字节流里，只是写入了字段号信息。另外，相对于xml和json来说，因为被编码成二进制，破解成本增大。

* 向后兼容：解析器遇到无法解析的字段，会自动跳过，不影响其它字段的解析，因此新增字段生成的字节流还是可以用旧版本的proto文件代码解析。

* 语言无关和平台无关：把proto文件转成字节流，可以采用不同的语言，比如后台用C++，客户端用Java或者Python等，字节流转具体对象可以根据需要自行选择语言。也就是说整个编解码过程完全不依赖某种语言的特性。

### 缺点

* 可读性差：因为最终是转成二进制流，不像xml和json能够直接查看明文。

## Protobuf使用场景

最重要的，技术最后还是得应用到具体的场景中，对于protobuf的使用场景，简单来说，业务要求命中其优点越多，缺点越少，就更能够使用Protobuf，比如说在某些场景对消息大小很敏感，或者传输的数据量不大，比如说APP登录场景，那么可以考虑使用Protobuf。
