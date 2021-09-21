---
title: Android签名机制原理与多渠道打包
date: 2019-11-23 20:59:26
tags:
---

## V1签名过程

随便找个apk解压，会看到有个META-INF文件夹，这个就是跟apk的签名有关的，里面会有三个文件：MANIFEST.MF，ANDROID.SF，ANDROID.RSA。

* MANIFEST.MF

	这个保存了apk中所有文件的消息摘要值，遍历apk所有的文件，对于每一个文件，都用SHA1（或者SHA256）计算其摘要，然后再做Base64编码，得到的值和文件路径组成key-value，写入到MANIFEST.MF文件中。示例如下：

		Manifest-Version: 1.0

		Name: AndroidManifest.xml
		SHA1-Digest: bVaU03v/s2D1ZnfNly3inexGx8E=
		
		Name: LICENSE-junit.txt
		SHA1-Digest: xa0kHcW0TK5p5IxSU7ej/t5BXqE=
		
		Name: LICENSE.txt
		SHA1-Digest: dTF36wNXk3NZRvp51lzE8FW7KYQ=
		
		Name: R/a/gu.xml
		SHA1-Digest: EImnskGmHG13zzjLQsBz7AaLVGU=
		
		Name: R/a/hf.xml
		SHA1-Digest: 1UI3hZQyzdEjUG/usTIhx245zDM=
		......

* CERT.SF

	这个是对MANIFEST.MF中的所有条目，再次进行SHA1+Base64处理，最终的结果同样组成key-value，写入到CERT.SF文件中。示例如下：

		Signature-Version: 1.0
		Created-By: 1.0 (Android)
		SHA1-Digest-Manifest: ktRWgoJhhmR9UKW2bkvJv9sWbwg=
		X-Android-APK-Signed: 2
		
		Name: AndroidManifest.xml
		SHA1-Digest: vPXU1jU9CV5ThOTQ1dhPspFhSDY=
		
		Name: LICENSE-junit.txt
		SHA1-Digest: i/XNa154J6oo2MWLxVtyjXAag2Q=
		
		Name: LICENSE.txt
		SHA1-Digest: dtxUfw7RZZWDxsyMx++B4WEugqM=
		
		Name: R/a/gu.xml
		SHA1-Digest: QgyDHCXLVsWQFsZVYbvZnimS81w=
		
		Name: R/a/hf.xml
		SHA1-Digest: N8tGKy5jd9e3TmOtLTmfwtUW1LM=
		......

* CERT.RSA

	首先这里需要一个证书，其实就是平时开发用到签名文件，然后利用这个证书里面的私钥对CERT.SF计算得到签名，然后把这个签名和证书写入到CERT.RSA中，注意写入的证书包含了公钥信息，私钥是不会写进去的。这里用到的证书是自签名的，也就是开发者自己生成，不用要求必须是CA机构发布的。

## V1验证过程

* 遍历apk中的所有文件，执行SHA1+Base64，判断能否跟MANIFEST.MF对应得上，对应不上，就说明校验失败。（这一步是为了判断apk是否被修改过）

* 遍历MANIFEST.MF中所有条目，执行SHA1+Base64，判断能否跟CERT.SF对应得上，对应不上，就说明校验失败。（这一步是为了判断MANIFEST.MF文件是否被修改过）

* 读取CERT.RSA里面的证书公钥和签名信息，用公钥对签名信息进行解密，得到的结果看是否跟CERT.SF对应得上，对应不上，就说明校验失败。（这一步是为了判断CERT.SF文件是否被修改过）

可以看到，这里会依次校验各个文件是否被修改过：apk --> MANIFEST.MF --> CERT.SF
假如修改了apk，MANIFEST.MF和CERT.SF也是能做对应的修改，但这时第三步校验会失败，因为CERT.SF已经被修改了，会导致校验失败，而我们没有私钥，无法计算出新的签名信息，因此CERT.RSA修改不了，除非重新签名。

对于非对称加密，有：

**公钥用于加密，私钥用于解密**

**私钥用于签名，公钥用于校验**

## V1签名举例

1. 下载微信的apk，解压

2. 找到MANIFEST.MF文件，找到AndroidManifest.xml文件的SHA1-Digest值为Lb4Rq2prbYpUiXh4uAbxGts4s74=。

3. 使用Hash.exe（下载链接[http://www.keir.net/hash.html](http://www.keir.net/hash.html "http://www.keir.net/hash.html")）计算AndroidManifest.xml文件的SHA1值为2DBE11AB6A6B6D8A54897878B806F11ADB38B3BE。

4. 去网站[https://the-x.cn/base64/](https://the-x.cn/base64/ "https://the-x.cn/base64/")计算这个SHA1值的Base64值，记得编码源格式要选HEX，结果为Lb4Rq2prbYpUiXh4uAbxGts4s74=，跟MANIFEST.MF中的一样。

5. 找到COM_TENC.SF文件，找到AndroidManifest.xml文件的SHA1-Digest值为c3JzQyDWuk4UK9Bzf2Z8RGootiM=。

6. 直接运行下面的代码

		String s = "Name: AndroidManifest.xml\r\nSHA1-Digest: Lb4Rq2prbYpUiXh4uAbxGts4s74=\r\n\r\n";
		MessageDigest messageDigest = MessageDigest.getInstance("SHA1");
		messageDigest.update(s.getBytes());
		byte[] digest = messageDigest.digest();
		System.out.println(new String(Base64.getEncoder().encode(digest), "ASCII"));

7. 最终打印c3JzQyDWuk4UK9Bzf2Z8RGootiM=，跟COM_TENC.SF中的一样。


## V1签名存在的问题

* 验证签名需要先解压，导致安装速度变慢。

* apk完整性校验不够，因为是基于单个文件做校验，只有在修改了文件内容才会导致校验失败，而修改apk的中央目录等信息后校验还是能校验成功。

因此，Android7.0开始，Google引入V2签名。

## V2原理

V2会直接对整个zip包进行计算生成签名信息，然后存储到zip的中央目录部分前面，因此在签名前和签名后的数据存放如下：

签名前：Contents of ZIP entries -- Central Directory -- End of Central Directory

签名后：Contents of ZIP entries -- APK Signing Block -- Central Directory -- End of Central Directory

Apk签名信息被做为zip的第二部分存放，只要zip被修改（不包括签名信息部分），就会导致签名校验失败。

可以看到V2解决了V1的两个问题，校验前不用解压apk了，而且只有修改apk就会导致校验失败，至于V2具体是怎么校验的，后面有空再研究。

## V1和V2兼容问题

对开发者来说，可以自己决定apk是采用V1还是V2签名，并不是说强制在Android N以上的都必须用V2，但为了避免V1上面提到的问题，建议是使用V2，而且说不定哪天Google就强制说必须用V2了。另外，如果apk的最小支持sdk在Android N之前，那就必须要有V1签名。

如果一个apk同时带了V1和V2签名，那么在V1的SF文件中就会有X-Android-APK-Signed: 2这个信息，表示该apk有V2签名，必须用V2签名校验，这么做的原因是为了避免破解者为了把V2签名删掉，想直接用V1签名。所以这里的保护机制为：V1保护V2，V2保护APK。

## 多渠道问题

如果采用V1签名，那么实现多渠道有三种方法：

* 采用Android Studio的productFlavors，可以在自定义一些变量值，从而能够区分不同apk包，但这个需要分别构建生成apk，效率低。

* 在META-INF目录下新建空文件，文件名就是渠道的唯一标识。META-INF的修改是不会影响到V1签名的，比如美团的Walle。

* 在zip的End of central directory record中的注释字段写入渠道信息，这样也不会影响到V1签名，比如腾讯的VasDolly。

如果采用V2签名，那么实现多渠道有两种方法：

* 同样还是能用Android Studio的productFlavors。

* 基于V2不会校验APK Signing Block这个区域的数据这个原理，在APK Signing Block中添加新的含渠道信息的键值对，同时修改End of central  directory record的中央目录偏移量。Walle和VasDolly都是采用这个方法实现V2签名多渠道打包的。


## 参考资料

[https://github.com/Tencent/VasDolly/](https://github.com/Tencent/VasDolly/ "https://github.com/Tencent/VasDolly/")
