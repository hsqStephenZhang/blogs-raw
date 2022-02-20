---
title: TLS
---

## 1. TLS 是什么

我们需要在网络中实现安全通信，最直接的思路是，将内容通过某种手段加密，再通过某种手段解密。

这个手段可以有两种：

1. 非对称加密。比如 RSA，ECC 算法，加解密过程是不可逆的，可以方便地对于数据进行签名/验证，但是计算速度慢
2. 对称加密。计算速度快，加密强度大，但关键是，加解密用的是相同的密钥，需要通过安全的手段传输密钥才行。

我们最终肯定还是希望可以使用性能更好的对称加密，但是协商对称加密密钥的过程，本身是不安全的，TLS 协议主要就是为了解决这个问题。

其实最关键的一步，就是从 0 到 1 的建立信任的过程，只要这个步骤做好了，之后就可以在已经加密的信道上面进行通信。

但是我们始终要明确一点：**网络是不安全的！** 不仅是你发出去的信息会被别人截获，甚至可能收到中间人发送的假消息，前者关注点在于**发送**，后者关注点在于**接收**。

针对发送和接收这两种情况，非对称加密分别有两个对策：

1. 接收方将公钥放在公开的地方，私钥自己好好保存，发送方需要使用公钥将数据加密，然后发送出去。这样，只有拥有私钥的接收方可以正确地解密；
2. 发送方使用私钥进行加密，只有通过公钥才能解密，并且，如果篡改了其中的内容，公钥解密是会失败的，这样，接收者也就可以接收到正确的消息了。

这两者都利用了非对称加密当中，**只有同时拥有公钥和私钥，才能进行解密** 这个先决条件，互为逆过程。

上面这两种方式中，都有一个很关键的问题：**公钥的安全传输！** 试想一下，如果公钥和私钥同时被他人截获并替换，那我们的信息就完全暴露在攻击者面前了。非对称加密又是建立在成功获取公钥的基础上，这两者的冲突似乎无法很好地解决！

```text
Tips:
  其实几乎所有的加密，都会建立在某些已经确定的信任链基础上，然后才能慢慢扩展。
```

为了解决这个问题，引入了根证书的概念。所谓根证书，是最最权威的机构颁发的认证，内置在操作系统当中，因此不会因为网络上的传输而无法验明真伪。而需要加密认证的网站，需要将自己的公钥上传，交给根证书认证机构，用该机构的密钥进行加密，作为 Certificate 颁发给网站申请者。

有访客访问网站的时候，会将这个 Certificate 下载到本地，并且根据系统内置的根证书链，对其内容验明真伪，如果为真，解密之后就可以获取到其中包含的网站公钥。

上面所说的 Certificate，就是真正的证书，目前几乎遵守 X.509 格式，内容大致如下：

```text
版本号
序列号
签名算法
颁发者
证书有效期
  开始日期
  终止日期
主题
主题公钥信息
  公钥算法
  主体公钥
颁发者唯一身份信息（可选）
主题唯一身份信息（可选）
扩展信息（可选）
签名
```

## 2. TLS 证书

SSL/TLS  有多种文件格式，主要的文件有：PEM, DER, PFX, JKS, KDB, CER, KEY, CRT, CSR，同时也会设计到几个协议，如 CRL, OCSP, SCEP  等，下面主要针对常用的文件格式和协议进行解释

### 2.1 introduction

其实上面的证书类型，笼统来说叫做类型，细分起来，区别还是很大的，可以大致划分为：

1. 编码格式
2. 文件扩展名

### 2.2 编码格式

编码格式可以分为 PEM 和 DER 两种，

1. PEM（private enhanced mail）

   ​ Openssl 默认使用 PEM 格式（文本）来存放各种信息，这是 openssl 默认的存放方式，一般 PEM 文件包含一下信息：

   - 内容类型。表示本文件存放的是什么内容的信息，以 `-----BEGIN XXX-----` 开始，以 `-----END XXX-----` 结束，XXX 可以是 `PRIVATE KEY` ，也可以是 `CERTIFICATE`，分别表示这是一个私钥还是公钥。
   - 头部信息。表示数据加密处理的方式，一般等同于加密算法+初始化向量。
   - 信息内容。BASE64 编码之后的数据。

2. DER（distinguished encoding rules）

   ​  使用二进制格式进行存储，对人类不友好。

两种编码可以互相进行转换：

1. PEM 转为 DER `openssl x509 -in cert.crt -outform der -out cert.der`
2. DER 转为 PEM: `openssl x509 -in cert.crt -inform der -outform pem -out cert.pem`

### 2.3 文件扩展名

这里是比较迷惑人的地方，虽然编码格式就 PEM 和 DER 两种，但是，文件扩展名千奇百怪。

1. CRT。certificate 的缩写，常见于 *NIX 系统，编码格式 unknown，如果为 PEM 编码，上面已经说明如何辨；
1. CET。还是 certificate，常见于Windows系统，同样编码格式 unknown，不过大多数应该是 DER 编码。和 CRT 是同义词；
1. KEY。这个含义应该很明确了，钥匙，可以是私钥，也可以是公钥，编码格式unknown；
1. PFX/P12。 对*NIX 服务器来说，一般 CRT 和 KEY 是分开存放在不同文件中的，但 Windows 则将它们存在一个 PFX 文件中（因此这个文件同时包含了证书及私钥）并且 PFX 通常会有一个“密码”，如果你想把里面的东西读取出来的话，它就要求你提供提取密码；
1. CSR。certificate signing request，即证书签名请求，这个并不是证书，而是向权威证书办法机构获取签名证书的申请，核心内容是一个公钥，在生成公钥的时候，同时还会生成一个相匹配的私钥，需要自己保管好。
1. JKS。Java Key Storage，这个是 Java 的专利，可以和 PFX 互相转换，但是和 Openssl 关系不大。

总结一下，CRT/CET 表示证书，Key 表示密钥（公钥/私钥），它们都可以表示为 PEM 和 DER 两种格式。很多时候人们并不会按照文件内容来添加后缀，有时候会很困惑，所以最好还是在文本编辑器中打开，然后自行查看。

## 3. TLS 建立加密通信过程

下面这个图片借鉴自网络，我觉得非常清晰，所以就直接粘贴了。

![SSL : TLS 握手过程](https://segmentfault.com/img/bVbCCMD)

TLS 握手的详细过程：

1. "client hello" 消息：客户端通过发送 "client hello" 消息向服务器发起握手请求，该消息包含了客户端所支持的 TLS 版本和密码组合以供服务器进行选择，还有一个 "client random" 随机字符串。
2. "server hello" 消息：服务器发送 "server hello" 消息对客户端进行回应，该消息包含了数字证书，服务器选择的密码组合和 "server random" 随机字符串。
3. 验证：客户端对服务器发来的证书进行验证，确保对方的合法身份，验证过程可以细化为以下几个步骤：
   1. 检查数字签名
   2. 验证证书链 (这个概念下面会进行说明)
   3. 检查证书的有效期
   4. 检查证书的撤回状态 (撤回代表证书已失效)
4. "premaster secret" 字符串：客户端向服务器发送另一个随机字符串 "premaster secret (预主密钥)"，这个字符串是经过服务器的公钥加密过的，只有对应的私钥才能解密。
5. 使用私钥：服务器使用私钥解密 "premaster secret"。
6. 生成共享密钥：客户端和服务器均使用 client random，server random 和 premaster secret，并通过相同的算法生成相同的共享密钥 KEY。
7. 客户端就绪：客户端发送经过共享密钥 KEY 加密过的 "finished" 信号。
8. 服务器就绪：服务器发送经过共享密钥 KEY 加密过的 "finished" 信号。
9. 达成安全通信：握手完成，双方使用对称加密进行安全通信。

参考资料：

1. [[HTTPS详解二：SSL / TLS 工作原理和详细握手过程](https://segmentfault.com/a/1190000021559557)]