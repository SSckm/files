## 摘要

> 本文主要介绍用wireshark来分析Https建立连接的过程。其中包括TCP三次握手流程以及TLS/SSL握手流程

![](https://img.soaer.com/blog/36.jpg)

## DNS解析

> 第四行：请求DNS服务器请求解析js.soaer.com的域名
>
> 第五行：DNS服务器返回了该域名所对应的IP地址为：39.106.117.20

## TCP的三次握手

### 第一次

>Flags设置SYN的值
>
>Sequence number:0, (序列号)还有其余的一些信息。

![](https://img.soaer.com/blog/37.jpg)

### 第二次

> Flags: 设置SYN和ACK
>
> acknowledgment number: 1 (有第一次握手中生成的序列号+1)
>
> Sequence number:0 返回序列号

![](https://img.soaer.com/blog/38.jpg)

###第三次

>Flags:设置ACK
>
>acknowledgment number: 1 (第二次握手中服务器生成的序列号+1)
>
>Sequence number:1 （有第一次握手中生成的序列号+1）返回序列号

![](https://img.soaer.com/blog/39.jpg)

> 至此三次握手成功。
>
> 流程图如下图所示：



![](https://img.soaer.com/blog/45.jpg)



# TLS/SSL握手过程

> 上面说了三次握手流程，https的数据是怎么传输的呢？https在经历了上面的三次握手后还需要哪些步骤才能建立加密数据通道呢？下面我们讲解一下Https的SSL/TLS协议

### Client Hello

> 主要信息如下：
>
> 1.Random随机信息（客户端第一次随机数据），用于后面密钥的生成
>
> 2.支持的加密算法列表（cipher suites）
>
> 3.TLS的版本信息
>
> 4.SessionID等等
>
> 5.压缩算法候选列表（compression methods）

![](https://img.soaer.com/blog/40.jpg)

### Server Hello 

> 主要信息如下：
>
> 1.Random服务端随机数据（服务端第一次随机数据）用于后面的密钥生成。
>
> 2.在客户端发送的加密列表里面选出一个加密组件（Client Hello阶段客户端发送了一组支持的加密组件）
>
> 3.选择加密算法。

![](https://img.soaer.com/blog/41.jpg)

###Server Certificate && Server Hello Done

> 主要信息如下：
>
> 1.服务器端配置对应的证书链，用于身份验证与密钥交换
>
> 2.通知客户端 server_hello 信息发送结束

![](https://img.soaer.com/blog/42.jpg)

### 客户端校验证书的合法性

> 1.证书链的可信性 trusted certificate path，知道校验到根证书
>
> 2.证书是否吊销
>
> 3.有效期 expiry date，证书是否在有效时间范围;
>
> 4.域名 domain，核查证书域名是否与当前的访问域名匹配，匹配规则后续分析;

### Client Key exchange

> 1.Client Key Exchange：客户端计算产生随机数字 Pre-master，并用证书公钥加密，发送给服务器
>
> 此时客户端已经得到了协商密钥所有组成部分（客户端第一次随机数。服务器第一次随机数，以及Pre-master）
>
> 2.Change Cipher Spec:客户端通知服务器后续的通信都采用协商的通信密钥和加密算法进行加密通信;
>
> 3.Encrypted Handshake Message：结合之前所有通信参数的 hash 值与其它相关信息生成一段数据，采用协商密钥 session secret 与算法进行加密，然后发送给服务器用于数据与握手验证

![](https://img.soaer.com/blog/43.jpg)

### 服务器验证数据

> 1.服务器用私钥解密加密的 Pre-master 数据，基于之前交换的两个明文随机数，计算得到协商密钥.至此服务器端也掌握了所有的密钥组成部分。
>
> 2.计算之前所有接收信息的 hash 值，然后解密客户端发送的 Encrypted Handshake Message，验证数据和密钥正确性;
>
> 3.验证通过之后，服务器同样发送 Change Cipher Spec 以告知客户端后续的通信都采用协商的密钥与算法进行加密通信;
>
> 4.Encrypted Handshake Message, 服务器也结合所有当前的通信参数信息生成一段数据并采用协商密钥 session secret 与算法加密并发送到客户端;

![](https://img.soaer.com/blog/44.jpg)



### 握手结束（可以发送应用加密数据Https至此握手全部完成）

> 客户端计算所有接收信息的 hash 值，并采用协商密钥解密 Encrypted Handshake Message，验证服务器发送的数据和密钥，验证通过则握手完成;

### 流程图如下：

![](https://img.soaer.com/blog/46.jpg)