---
title: APK签名机制
categories:
  - Android进阶
tags:
  - apk
  - 签名
date: 2017-01-31 18:50:34
---

本文介绍apk的签名机制。

<!-- more -->


# 一、APK签名

apk发布者需要使用android 密钥生成工具创建的keystore对apk进行签名。

# 二、APK升级

用于升级的新版本的apk必须使用原有的keystore进行签名，这样才能保证顺利升级。

# 三、android签名文件

采用RSA密钥机制对APK进行签名。
对比已签名和未签名的apk包，签名的apk包多了一个名为META-INF的文件夹，里面有三个文件：MANIFEST.MF、CERT.SF和CERT.RSA。这三个文件就是签名相关的文件。

## 3.1 MANIFEST.MF

遍历apk包中所有文件，对非文件夹、非签名文件的文件，逐个生成SHA1数字签名信息，再用Base64进行编码。之后生成的签名写入MANIFEST.MF文件。

## 3.2 CERT.SF

对生成的MANIFEST.MF签名信息，使用SHA1-RSA算法，用私钥进行签名。

## 3.3 CERT.RSA

在CERT.RSA文件中保存公钥、所采用的加密算法等信息。

# 四、Android签名验证

android原本的签名验证是怎么样的呢？在安装过程中，系统会读取apk包中的MANIFEST.MF、CERT.SF和CERT.RSA：

1. 用MANIFEST.MF验证apk包的完整性。
2. 用CERT.SF验证MANIFEST.MF的完整性。
3. 用CERT.RSA验证CERT.SF的签名。

# 五、总结APK的签名流程

Apk包加密的公钥就打包在apk包内，且不同的私钥对应不同的公钥。公钥一致，则私钥一致。可以通过公钥的对比来判断私钥是否一致。(公钥与私钥是一一对应的关系，这种对应关系是怎么形成的？？谁分配私钥公钥。Answer：是由加密算法生成的)

# 六、tool instruction

- Keytool是一个java数据证书的管理工具。
- jarsigner  是apk签名工具。

jdk包提供以上工具，需自行安装jdk包。

# 七、keystore生成

keystore就是用来保存密钥对的，比如公钥和私钥。用jdk的keytool工具生成keystore：

```
keytool -genkey -alias test.keystore -keyalg RSA -validity 20000 -keystore test.keystore 
```

![keystore-generate.png](/img/archives/keystore-generate.png)

Note：
- alias test.keystore 生成别名为test.keystore的keystore
- keyalg RSA 加密和数字签名的算法
- validity 20000 有效天数


<font color="red">输入keystore password后经过加密算法和加密工具生成私钥和公钥，要保护好password，因为只能用password打开keystore，然后访问key。

用私钥证书签名，用公钥证书验证。输入密码和developer信息后，使用一些算法生成相应的数字证书，其他人可以用这个数字证书来验证这个apk是不是该developer开发的。

# 八、证书的生成

通过.keystore文件生成*.cer证书文件:

```
Keytool -export -keystore test.keystore -alias test.keystore -file test.cer
```

![keystore-export.png](/img/archives/keystore-export.png)

Note：
将把证书库 test.keystore 中的别名为 test.keystore的证书导出到 test.cer 。证书文件中，它包含证书主体的信息及证书的公钥，不包括私钥，可以公开。 

# 九、使用Keystore 签名apk

使用jarsigner工具对apk进行签名：

```
jarsigner -verbose -keystore test.keystore  -signedjar HelloWorld_sign.apk Hello_un_sign.apk test.keystore
```

![apk-sign.png](/img/archives/apk-sign.png)

Note：
- keystore  test.keystore指定密钥库的位置
- signedjar  HelloWorld_signed.apk HelloWorld_unsign.apk test.keystore  依次是签名后的文件HelloWorld_signed.apk，要签名的文件HelloWorld_unsign.apk和密钥库test.keystore  

# 十、相关问题

## 10.1 证书有效期

只在程序安装时才检查证书的有效期：
- 证书的有效期包含程序的生命周期，只在程序安装时才检查证书的有效期.
- 一旦证书失效，程序将不能正常升级，但已安装的程序可以继续使用；
- Android Market强制要求所有应用程序证书的有效期要持续到2033年10月22日以后。

已过期的key是无法再对apk进行签名的.

## 10.2 对已签名Apk的再签名

- 追加签名.
- 升级时,需要验证所有签名一致.

## 10.3 平台里的证书

- 平台并没有特别存放证书.
- 当系统扫描系统apk,发现是system uid 时,就会将其签名再存放到 system uid对应的签名中.(一对一)
- 当用户在此安装apk使用system uid 时, 就会通过system uid 索引找到签名进行对比.




