---
title: Android系统build阶段签名机制
categories:
  - Android进阶
tags:
  - 签名
  - android
date: 2017-01-31 15:48:29
---

本文介绍Android系统build阶段的签名机制。

<!-- more -->

# 一、系统build阶段签名机制

1、系统中有4组key用于build阶段对apk进行签名：

- Media
- Platform
- Shared
- Testkey

default key是放在Android源码的/build/target/product/security目录下:
- media.pk8与media.x509.pem；
- platform.pk8与platform.x509.pem；
- shared.pk8与shared.x509.pem；
- testkey.pk8与testkey.x509.pem；

其中，`*.pk8`文件为私钥，`*.x509.pem`文件为公钥，这需要去了解非对称加密方式。

2、在apk的android.mk文件中会指定LOCAL_CERTIFICATE 变量：

LOCAL_CERTIFICATE可设置的值如下：   

``` 
LOCAL_CERTIFICATE := testkey   # 普通APK，默认情况下使用
LOCAL_CERTIFICATE := platform  # 该APK完成一些系统的核心功能,这种方式编译出来的APK所在进程的UID为system
LOCAL_CERTIFICATE := shared    # 该APK是media/download系统中的一环
LOCAL_CERTIFICATE := media     # 该APK是media/download系统中的一环
```

如果不指定，默认使用testkey。

对应的，除了在Android.mk指定上述的值，还需要在APK源码的AndroidManifest.xml文件的manifest节点里面申明权限：

```
android:sharedUserId="android.uid.system"
android:sharedUserId="android.uid.shared"
android:sharedUserId="android.media"
```

3、Build规则是Build/core/prebuilt.mk。
4、在build/core/config.mk中，`DEFAULT_SYSTEM_DEV_CERTIFICATE`可以通过`PRODUCT_DEFAULT_DEV_CERTIFICATE`去指定各家厂商的key path。

我们可以看到，默认为build/target/product/security/testkey。
```
# The default key if not set as LOCAL_CERTIFICATE
ifdef PRODUCT_DEFAULT_DEV_CERTIFICATE
  DEFAULT_SYSTEM_DEV_CERTIFICATE := $(PRODUCT_DEFAULT_DEV_CERTIFICATE)
else
  DEFAULT_SYSTEM_DEV_CERTIFICATE := build/target/product/security/testkey
endif
```

# 二、自定义系统签名的key

上面介绍了系统有默认四组key，那么如果我们要制作自己的key，需要怎么做呢？
在build/target/product/security/目录下有一个README，里面有说明怎么制作这些key并且使用。

1、进入Development/tools/ 目录 

![cd-dev-tool.png](/img/archives/cd-dev-tool.png)

2、使用make_key工具生成签名文件：

```
sh make_key releasekey  '/C=CN/ST=Guangdong/L=Shenzhen/O=Mediatek/OU=MTK/CN=fzll/emailAddress=maoao530@foxmail.com'
```

其中：
- C  : Country Name (2 letter code)
- ST : State or Province Name (full name)
- L  : Locality Name (eg, city)
- O  : Organization Name (eg, company)
- OU : Organizational Unit Name (eg, section)
- CN : Common Name (eg, your name or your server’s hostname)
- emailAddress : Contact email address

3、用ls命令发现目录下多了两个文件：releasekey.x509.pem 和 releasekey.pk8

![release-key.png](/img/archives/release-key.png)

4、同样的步骤生成platform / shared /  media 

5、用自定义的key替换build/target/product/security/目录下面的key。

# 三、对APK进行系统签名

为了使apk有system权限，通常我们需要对其进行系统签名：

1、在应用程序的AndroidManifest.xml中的manifest节点中加入

```
android:sharedUserId="android.uid.system"这个属性。
```

2、修改它的Android.mk文件，加入

```
LOCAL_CERTIFICATE := platform
```

重新编译，生成的apk就有修改system权限了,我们通过`ps`命令查看APK所在进程的UID，发现值为system。

