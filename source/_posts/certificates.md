---
title: HTTPs相关
comments: true
toc: true
date: 2018-07-20 11:51:21
categories: Others
tags: 
    - CA
    - encryption
---

```sh
CA Certificate Authority 认证授权机构
cert Certificate 证书
csr Certificate Signing Request 证书签名申请
pk public key
psk pre-shared key
encryption 加密
decryption 解密
symmetric encryption 对称加密
asymmetric encryption 非对称加密
```

<!--more-->

# CA证书

CA证书顾名思义就是由CA机构发布的数字证书。

包含有：

- 颁发者
- 使用者
- 版本
- 签名算法
- 签名哈希算法
- 使用者
- 公钥
- 指纹
- 指纹算法
- ……

证书的认证是安装证书链执行的，证书链的意思是有一个证书机构A，A生成证书B，B也可以生成证书C，A证书或者certification authority of wosign证书在整个证书链上就被称为根证书。证书验证的机制是只要根证书是受信任的，那么它的子证书都是可信的。如果一个证书的根证书是不可信的，那么这个证书肯定也是不可信任的。

中国的安全认证体系分为金融CA和非金融CA。在金融CA方面，根证书由中国人民银行管理，在非金融CA方面，由中国电信负责。中国CA又可分为行业性CA和区域性CA,行业性CA中影响最大是中国金融认证中心和中国电信认证中国；区域性CA主要是以政府为背景，以企业机制运行，其中广东CA中心和上海CA中影响最大。

# SSL与TLS

https就是http加SSL，SSL（security sockets layer，安全套接层）是为网络通信提供安全及数据完整性的一种安全协议，SSL协议位于TCP/IP协议与各种应用层协议（包括http）之间，SSL3.0版本以后又被称为TLS (transport layer security，传输层安全 )。

# 对称加密与非对称加密

1. 对称加密就是发送双发使用相同的密钥对消息进行加解密。

    常见的对称加密为DES、3DES、AES等。

2. 非对称加密是发送双方各自拥有一对公钥私钥，其中公钥是公开的，私钥是保密的。当发送方向接收方发送消息时，发送方利用接收方的公钥对消息进行加密，接收方收到消息后，利用自己的私钥解密就能得到消息的明文。

    其中非对称加密方法有RSA、Elgamal、ECC等。

解决第三方冒充的方法是将发送内容进行二次加密，并且通讯双方有可靠的途径知道对方的公钥:

- A发送给B时候，先用A的私钥加密，然后再用B的公钥加密。
- B收到后，先用B的私钥解密，再用A的公钥解密，得到明文。

获取公钥的可靠途径不一样，就可以有不容的实现方式：

1. 通信双方事先有对方的公钥，这种方法比较麻烦，要面对面交换。显然不适合大规模应用，用在夫妻之间到是比较好的！

2. 第三方的数字签名，这个就比较好了，大家都把公钥放在第三方CA那里，通信发起方问CA要双方的公钥，并传给对方。

HTTPS 采用混合的加密机制，使用非对称密钥加密用于传输对称密钥来保证安全性，之后使用对称密钥加密进行通信来保证效率。（下图中的 Session Key 就是对称密钥）

![How HTTPs works](/images/How-HTTPS-Works.png)

服务器的运营人员向 CA 提出公开密钥的申请，CA 在判明提出申请者的身份之后，会对已申请的公开密钥做数字签名，然后分配这个已签名的公开密钥，并将该公开密钥放入公开密钥证书后绑定在一起。

进行 HTTPs 通信时，服务器会把证书发送给客户端。客户端取得其中的公开密钥之后，先使用数字签名进行验证，如果验证通过，就可以开始通信了。

通信开始时，客户端需要使用服务器的公开密钥将自己的私有密钥传输给服务器，之后再进行对称密钥加密。

![Sign and verify](/images/ca.png)

# 数字签名与签名认证 digital signature

发方将原文用哈希算法求得数字摘要，用签名私钥对数字摘要加密得数字签名，发方将原文与数字签名一起发送给接受方。
数字签名的操作过程需要有发方的签名数字证书的私钥及其验证公钥。具体过程如下：首先是生成被签名的电子文件（《电子签名法》中称数据电文），然后对电子文件用哈希算法做数字摘要，再对数字摘要用签名私钥做非对称加密，即做数字签名；之后是将以上的签名和电子文件原文以及签名证书的公钥加在一起进行封装，形成签名结果发送给收方，待收方验证。

接收方收到数字签名的结果，其中包括数字签名、电子原文和发方公钥，即待验证的数据。接收方进行签名验证。验证过程是：接收方首先用发方公钥解密数字签名，导出数字摘要，并对电子文件原文作同样哈希算法得一个新的数字摘要，将两个摘要的哈希值进行结果比较，结果相同签名得到验证，否则签名无效。这就作到了《电子签名法》中所要求的对签名不能改动，对签署的内容和形式也不能改动的要求。

## 摘要 digest

OpenSSL支持的摘要算法

- md5
- md4
- mdc2
- sha1
- sha
- sha224
- ripemd160
- dss1
- whirlpool

## 签名 signature

签名的一般过程：先对数据进行摘要计算，然后对摘要值用私钥进行签名。

### RSA私钥签名验签

```sh
#生成RSA密钥对
$ openssl genrsa -out rsa_private.key

#由公钥导出私钥
$ openssl rsa -in rsa_private.key -pubout -out rsa_public.key

#用RSA私钥对SHA1计算得到的摘要值`签名`
$ openssl dgst -sign rsa_private.key -sha1 -out sha1_rsa_file.sign file.txt

#用相应的公钥和相同的摘要算法进行`验签`，否则会失败
$ openssl dgst -verify rsa_public.key -sha1 -signature sha1_rsa_file.sign file.txt

#也可以使用相同的私钥和相同的摘要算法进行验证
$ openssl dgst -prverify rsa_private.key -sha1 -signature sha1_rsa_file.sign file.txt
```

### DSA密钥签名验签

```sh
#生成DSA参数
$ openssl dsaparam -out dsa.param 1024

#由DSA参数产生DSA私钥
$ openssl gendsa -out dsa_private.key dsa.param

#由DSA私钥生成DSA公钥
$ openssl dsa -in dsa_private.key -out dsa_public.key -pubout

#用DSA私钥对SHA384计算的摘要值进行签名。
$ openssl dgst -sign dsa_private.key -sha384 -out sha384_dsa.sign file.txt

#用相应的公钥和摘要算法进行验签
$ openssl dgst -verify dsa_public.key -sha384 -signature sha384_dsa.sign file.txt

#用相同的私钥和摘要算法验签
$ openssl dgst -prverify dsa_private.key -sha384 -signature sha384_dsa.sign file.txt
```

>DSA在每次签名时,使用了随机数k,如果对同一消息进行多次签名,签名结果是不同的,所以DSA是一种随机式数字签名。

# SSL握手 handshake

- 客户端访问服务器,发送ssl版本、客户端支持的加密算法、随机数等消息。
- 服务器向客户端发送ssl版本、随机数、加密算法、证书（证书出现了）等消息。
- 客户端收到消息后，判断证书是否可信（如何判断可信，看下文介绍），若可信，则继续通信，发送消息包括：向服务器发送一个随机数，从证书中获取服务器端的公钥，对随机数加密；编码改变通知，表示随后信息都将使用双方协定的加密方法和密钥发送；客户端握手结束通知。
- 服务器端对数据解密得到随机数，发送消息：编码改变通知，表示随后信息都将使用双方协定的加密方法和密钥发送。

非对称加密算法对数据加密非常慢，效率低，而对称加密加密效率很高，因此在整个握手过程要生成一个对称加密密钥，然后数据传输中使用对称加密算法对数据加密。