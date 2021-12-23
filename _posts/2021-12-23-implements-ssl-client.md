# SSL客户端开发

之前对密码学有一定程度的研究，这次本着get your hand dirty的原则，打算造一造轮子——一个简单的SSL客户端。项目代码见</br>
SSL/TLS最广泛的应用当然是HTTP，所以一开始拿OkHttp的源码研究了一下。不久后发现，OkHttp是比较上层的，高层逻辑比较多，翻来翻去，翻到这份[JSSE参考指南](https://docs.oracle.com/javase/7/docs/technotes/guides/security/jsse/JSSERefGuide.html)。</br>
JSSE参考指南包含API框架和相应的实现，但是底层实现部分相关的资料比较少，对着几份官方的文档和Wireshark抓的包，整理了一下SSLSocket实验结果。

## 证书准备

## 服务端开发
简单地配置了Spring Boot Web（https），证书按需要安装单证书（相当于自签证书）或fullchain证书。

## 原理
![SSL/TLS握手](https://github.com/lvv9/lvv9.github.io/blob/master/pic/image_2021-12-24_00-19-09.png?raw=true)</br>
从上图可以看到，TLS协议第一个安全方面的验证，就是证书的验证
> The server sends the client a certificate or a certificate chain. A certificate chain typically begins with the server's public key certificate and ends with the certificate authority's root certificate.


就是说，证书的验证，是按证书链来验证的，如果服务端只安装单证书，会发生：
