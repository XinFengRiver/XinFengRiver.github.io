---
layout: default
title:  "TLS安全套件的简介与实践"
description: ""
date: 2020-01-06 6:00:00 +0800
categories: fend
---

HTTP协议传输的文本是明文的，任何中间代理都可以获悉client与server传输的信息，毫无保密性可言。并且HTTP协议没有校验通信双方的身份，也没有校验数据是否被篡改过，这就意味着中间代理可以冒充服务器或客户端，向双方发出错误的指令。SSL/TLS设计的目的就是解决这些安全问题。

SSL/TLS发展历程比较复杂，可以将TLS协议看作是SSL协议的升级版。本文介绍的是TLS协议相关的安全套件，包括基本的概念和通信的流程，但不涉及相关的加密算法，所以细节方面一定是不足的。在文章末尾，我会讲解如何使用`openssl`和`nginx`搭建简单的抓包实验，以加深读者对整个TLS通信流程的理解。

## 术语
这些术语对理解数据的加解密十分关键，我先简单地介绍这些概念，随着文章的深入你会理解它们，建议先大致浏览一遍，后文提到了再回顾。
- **密码**：不同于我们日常理解的"密码"（应当称之为"口令"），专业上的"密码"是一种算法，相当于function：`密码(明文) = 密文`。比如，我们把用户的口令存储为`md5(口令)`，这里的`md5()`就是密码。
- **密钥**：对于同一个密码（加密算法），为了区分不同的实例，还得引入新的变量"密钥"，那么加密的function是这样的：`密码(密钥，明文) = 密文`。
- **对称秘钥加密**：使用同一个密钥，既可以将明文加密为密文、也可以将密文解密为明文。听上去是不是有些不可思议，其实它的基本原理是简单的异或操作，感兴趣的读者可以自行搜索。
- **非对称密钥加密**：使用不同的密钥A和B，A可以解密用密钥B加密的密文，反过来，B也可以解密用密钥A加密的密文。但是，A无法解密密钥A加密的密文，B也无法解密密钥B加密的密文。常用的非对称密钥加密算法是RSA。
- **公开密钥加密**：字面意思，就是使用公开的密钥加密。公开密钥加密一般与非对称密钥加密一起用，例如公开密钥A并称之为**公钥**，密钥B称之为**私钥**，那么使用公钥加密的数据，除了掌握私钥的人谁也无法解密。
- **哈希运算**：将一大段数据映射为一小串数据，就可以用这串占用空间小的数据代表这一大段数据了，适合用来验证数据是否被修改了（数据的完整性）。由于哈希运算的规则相当复杂，使得逆运算几乎不可行，冲突基本不会发生。例如`hash(A) -> B`，很难将B还原为A ，也很难找到一个C满足`hash(C) -> B`。常用的哈希算法有`md5`、`sha`。

## TCP/IP模型中的SSL/TLS
按照通用的说法，SSL/TLS在TCP/IP模型中的位置位于运输层和应用层中间，负责将加密的应用层信息交由运输层。SSL/TLS包括握手层（Handshake Layer）和记录层（Record Layer），前者负责客户端与服务端正式通信前的协商工作，后者则负责正式通信中的数据加解密。

![TCP/IP模型中的SSL/TLS](/assets/image/posts/tls-intro-and-practice/tls-in-tcpip-model.jpeg){: .img-center width="500px"}

就了解SSL/TLS而言，记录层没有太多值得说的内容，我们把目光聚焦在握手层。握手层包括三个协议，分别是握手协议（Handshake Protocol）、密钥规格变更协议（Change Cipher Spec Protocol）、警报协议（Alert Protocol）。

## TLS安全套件的组成
针对HTTP协议的安全缺陷，TLS安全套件提供了一套组合拳来保护客户端与服务端的整个通信流程。这些套件包括：
1. 身份验证算法。在TLS握手阶段，用于客户端验证服务端的身份（在某些应用场景服务端也要验证客户端的身份）。
2. 加密算法。解决通信过程中数据的保密问题，在TLS套件中用的是对称密钥加密算法。
3. 密钥交换算法。对称密钥加密算法需要双方约定一个对称密钥，这个算法就是用于协商密钥的。对应图中的`AES_128_GCM`。
4. 签名hash算法。用于TLS握手期间，防止身份验证和密钥交换的数据被篡改。对应图中的`SHA256`。

![TLS安全套件的组成](/assets/image/posts/tls-intro-and-practice/tls-cipher.png){: .img-center width="400px"}

## TLS的握手阶段

TLS的整个通信流程可以分为握手过程和通信过程，分别对应刚刚所述的分层模型中的握手层和记录层。握手过程应用了握手层的三个协议，涉及安全套件协商、证书发送（身份验证）、密钥交换、密钥更换、警报等动作。在TCP三次握手成功后，客户端和服务端会开始协商安全套件，客户端会发起一个`client hello`，内容是客户端支持的安全套件列表以及这些安全套件相关的必要信息。服务端需要选择支持的安全套件并取用相应的数据，将选用的结果告知客户端。为了减少通信的次数，服务端返回协商结果的同时会发送签名证书和交换DH密钥。

![TLS握手过程和通信过程](/assets/image/posts/tls-intro-and-practice/tls-process2.jpeg){: .img-center width="400px"}

## 服务端身份验证——发送签名证书
TLS协议要解决的第一个问题就是如何让客户端在依赖条件最少的情况下验证服务端的身份，基于非对称加密技术的公钥基础设施是目前的解决方案。

### 数字证书、数字签名与验签
如果你点击浏览器地址栏左边的https安全标识，会看到数字证书，仔细浏览，可以发现证书里面有组织信息、证书时效、公钥信息、指纹信息等等。

![签名证书](/assets/image/posts/tls-intro-and-practice/tls-cert.png){: .img-center width="400px"}

这本数字证书是怎么来的呢？在开发者生成自有的私钥和公钥之后，需要将组织、公钥、一些拓展信息提交给证书登记机构，证书颁发机构(CA)会使用机构的私钥加密这些证书信息（签名），最终得到一个签名的证书。这本证书最后会存放到两个地方：开发者的服务器和证书机构的OCSP在线查询服务，前者传递证书给客户端，后者用于校验证书的有效性。

签名和验签的流程如下：
![数字签名与验签流程](/assets/image/posts/tls-intro-and-practice/tls-digital-sign.png){: .img-center width="600px"}
1. CA会使用一个哈希函数将对象提交的数据（也就是证书里的信息）序列化成哈希值A，并使用CA的私钥加密，加密的过程称为"签名"，加密后的值称为"**数字签名**"，数字签名会附加到证书上。这一步骤使用的算法一般是`sha1WithRSAEncryption`，看名字就知道，`sha1`就是哈希算法，`RSA`就是非对称加密算法。
2. 客户端收到返回的数据后，会取出数字签名并使用证书的公钥解密数据，得到那串哈希值A。客户端的系统一般会内置一些CA机构的根证书，客户端可以通过"**证书信任链**"的机制验证证书的有效性。
3. 接着，客户端会用同样的哈希算法对证书数据加密，得到一串哈希值B。只要哈希值A与哈希值B相同，就可以认为服务端返回的证书的确是由可信赖的CA机构颁发的，没有被中间代理篡改。
4. 最后，客户端可以从证书中取出服务端提供的公钥，在整个握手阶段客户端都会使用公钥加解密数据。

有心的读者会发现，这个流程的安全基础是证书信任链，这就不得不提公钥基础设施了。

### 公钥基础设施
由证书登记机构、颁发机构以及颁发机构提供的查询服务构建的体系，我们称之为PKI(Public Key Infrastructure，公钥基础设施)。客户端通过链式追踪的方式验证数字证书，这个过程有点像DNS的递归查询过程。客户端收到数字证书X后，会读取数字证书中的签发机构A，通过根证书的签发机构验证机构A是否有效，如果有效则信机构A，再从机构A验证数字证书X是否有效，有效则信任数字证书X，并使用证书附带的公钥解密。

![数字签名与验签流程](/assets/image/posts/tls-intro-and-practice/tls-public-key-search.png){: .img-center width="500px"}

这两张图表明了`docs.microsoft.com -> Microsoft IT TLS CA 1 -> Baltimore CyberTrust Root`这一整个证书信任链的过程。

![签名证书](/assets/image/posts/tls-intro-and-practice/tls-cert.png){: .img-center width="400px"}

![根证书](/assets/image/posts/tls-intro-and-practice/tls-root-cert.jpg){: .img-center width="400px"}

客户端查询证书信任链时访问的服务是OCSP在线查询服务。PKI与服务端签名、客户端验签的整个流程如这张图所示：

![数字签名与验签流程](/assets/image/posts/tls-intro-and-practice/pki.png){: .img-center width="500px"}

至此，我们完成了TLS握手阶段身份验证的流程，客户端拿到了服务端提供的公钥，这个公钥可以为握手阶段的DH密钥交换提供加密保障。

## 通信过程加密——AES算法与DH密钥交换
握手阶段除了身份验证，还需要协商后面的通信过程用的加密算法。为什么我们不直接用RSA公私钥加解密呢？主要是这两个原因：
1. RSA算法加解密的时间比较长，不适合用于频繁的数据通信。
2. 私钥泄漏后通信的历史数据将失去保密性。虽然在RSA的加持下破解私钥的成本很高、用时很长，但必定会被未来计算性能更高的计算机破解。如果通信过程也使用了这对公私钥，那么私钥泄漏就相当于公开了历史的所有通信数据，用业界术语描述就是没有"前向保密性"。

我们可以想象到通信过程用的加密算法必须克服以上的两个缺点，也就是加密效率高、具备前向保密性。针对加密效率，TLS安全套件用了对称加密技术，常用的算法如AES算法，加密的速度明显高于RSA，其原理比较复杂，我不作讲解，不会影响你了解通信加密过程；针对前向保密性，TLS安全套件使用了DH密钥交换的技术。

### DH密钥交换
AES算法仅用到一个密钥，供服务端和客户端共同使用，使用这个密钥的可以实现双向的加解密。为了实现前向保密性，这个密钥必须是临时的，只能短期内使用，DH密钥交换技术就是用来生成临时密钥的，TLS1.2中DH密钥交换的流程如下：

![DH密钥交换](/assets/image/posts/tls-intro-and-practice/tls-dh-exchange.png){: .img-center width="550px"}

1. 服务端确定了选用的安全套件后，会生成一对临时的公私钥A1和B1（跟RSA算法的公私钥不一样），然后将其中的公钥B1返回给客户端。
2. 客户端获悉选用的安全套件后，也会生成一对临时的公私钥A2和B2，并使用私钥A2和接收到的公钥B1计算得到密钥C，然后将公钥B2发送给服务端。
3. 服务端接收到公钥B2后，用它与私钥A1计算得出密钥C。没错，这个密钥C与客户端计算得到的密钥C是同一个密钥。

经过这三步，客户端和服务端就具备了一样的密钥，使用AES算法对双方发送的数据加解密了。

值得一提的是，交换密钥的过程为了防止中间人攻击（即中间代理自己生成一对公私钥，分别向客户端和服务端发送公钥，从而生成两个密钥，它们分别跟客户端与服务端的密钥相同，具备解密两方数据的能力），服务端会使用自有的私钥为交换密钥的数据签名以验证身份，签名过程类似CA证书签名的过程。

## demo演示：生成公私钥以及TLS握手
### 使用nginx搭建web服务
nginx的安装方法我就不介绍了。假设我们要访问的域名是`www.test.com`，我们可以在配置文件`nginx.conf`写一个简单的匹配：
```
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    server {
        listen 80;
	    server_name www.test.com;
    }

    location / {
        root /Users/andrew/nginx/www;
        index  index.html index.htm;
    }
}
```
在http模块，我配置了一个server，域名是www.test.com，监听80端口，location匹配该域名下所有资源路径的请求，返回的本地`/Users/andrew/nginx/www`文件夹下的文件`index.html`（需要自己新建）。重载nginx使配置生效：
```
nginx -s reload
```
接着，打开本地的hosts文件修改域名和ip的映射：
```
127.0.0.1 www.test.com
```
打开浏览器输入域名，访问成功。

如果我们尝试访问`https://www.test.com`，浏览器就会提示无法访问（`ERR_CONNECTION_REFUSED`），这是因为服务端没有配置https，无法处理客户端发起的SSL/TLS的握手协商。在配置https之前，我们先生成所需要的密钥和证书。

### 生成公私钥与证书——OpenSSL
有一类证书称为"自签证书"，顾名思义，就是开发者自己签发的证书。这类证书浏览器是不信任的，需要使用者在系统配置将证书设置为可信任。一般来说，公司为了避免泄漏生产环境的密钥，都会在测试环境使用自签证书。生成私钥和自签证书的命令如下：

```bash
# 生成私钥
openssl genrsa -out test.com-key.pem

# 生成证书申请文件
openssl req -new -key test.com-key.pem -out test.com.csr

# 生成自签证书
openssl x509 -req -in test.com.csr -out test.com.crt -signkey test.com-key.pem -days 3650
```

我找到了OpenSSL的指令手册[《OpenSSL commands》](https://www.openssl.org/docs/manmaster/man1/){:target="_blank"}和一篇介绍生成自签证书的文章[《Openssl生成自签名证书，简单步骤》](https://ningyu1.github.io/site/post/51-ssl-cert/){:target="_blank"}，感兴趣的读者可以看看。

### nginx配置TLS
我们修改`nginx.conf`以支持https：

```
worker_processes  1;
events {
    worker_connections  1024;
}

http {
    server {
	listen 80;
	listen 443 ssl;
	server_name www.test.com;
	ssl_certificate test.com.crt;
	ssl_certificate_key test.com-key.pem;
	ssl_protocols TLSv1.2 TLSv1.1 TLSv1;
	ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4;
	ssl_prefer_server_ciphers on;

        location / {
            root /Users/andrew/nginx/www;
            index  index.html index.htm;
        }
    }
}
```
可以看到，sever配置增加了监听的端口`443`，并标识了`ssl`表明是SSL/TLS通信的端口。`ssl_certificate`和`ssl_certificate_key`分别是证书文件和私钥文件，`ssl_protocols`指令是支持的安全协议，优先级从左到右。`ssl_ciphers`指令是安全套件的组成，拿`ECDHE-RSA-AES256-GCM-SHA384`举例，`ECDHE`是密钥交换的算法，`RSA`是身份认证的算法，`AES256-GCM`是通信加密的算法，`SHA384`是验证数据完整性的算法。`ssl_prefer_server_ciphers`开启后，SSL/TLS的协商过程会按服务端配置的顺序选用`ssl_ciphers`罗列的安全套件。

配置完成后重载nginx配置，访问`https://www.test.com`，浏览器依然会提示证书有风险（因为自签证书没在CA机构注册，验证结果是无效），但是用户可以选择继续浏览。

![请求的自签证书](/assets/image/posts/tls-intro-and-practice/tls-self-signed.png){: .img-center width="400px"}

### 抓包看协商
最后，我们从抓到的包数据直观地看看协商过程。抓包工具比较多，我用的是wireshark，因为wireshark不知道握手阶段协商的对称密钥，需要将密钥暴露给wireshark才可以，配置的方法参考这篇文章[《使用 Wireshark 调试 HTTP/2 流量》](https://imququ.com/post/http2-traffic-in-wireshark.html){:target="_blank"}。
配置好HTTPS的解密之后，在抓包配置中选中回环网卡(对应127.0.0.1)、在过滤器中输入域名配置：

![wireshark配置](/assets/image/posts/tls-intro-and-practice/wireshark-config.png){: .img-center width="800px"}

#### 自签证书域名的https抓包结果

![server hello](/assets/image/posts/tls-intro-and-practice/wireshark-client-server.png){: .img-center width="800px"}
我在图中标注了本次请求的TLS握手阶段和通信阶段。

从客户端发出的协商内容，可以看到客户端支持的协议版本和可供选择的安全套件。

![client hello](/assets/image/posts/tls-intro-and-practice/wireshark-client-hello.png){: .img-center width="500px"}

服务端选用的安全套件(server hello)、证书、交换密钥hello done是一同发出的。

![server hello](/assets/image/posts/tls-intro-and-practice/wireshark-server-hello.png){: .img-center width="500px"}

展开证书可以看到相关信息，我标注了服务端给的RSA算法用的公钥、签名时用的哈希算法。

![server hello](/assets/image/posts/tls-intro-and-practice/wireshark-server-cert.png){: .img-center width="500px"}

展开交换密钥的部分：

![server hello](/assets/image/posts/tls-intro-and-practice/wireshark-server-dh-exchange.png){: .img-center width="500px"}

--- DONE ---
