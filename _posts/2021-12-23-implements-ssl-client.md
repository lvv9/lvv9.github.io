# SSL客户端开发验证

之前对密码学有一定程度的研究，这次本着get your hand dirty的原则，打算造一造轮子——一个简单的SSL客户端。项目代码见<br>
SSL/TLS最广泛的应用当然是HTTP，所以一开始拿OkHttp的源码研究了一下。不久后发现，OkHttp是比较上层的，高层逻辑比较多，翻来翻去，翻到这份 [JSSE参考指南](https://docs.oracle.com/javase/7/docs/technotes/guides/security/jsse/JSSERefGuide.html) 。<br>
JSSE参考指南包含API框架和相应的实现，但是底层实现部分相关的资料比较少，对着几份官方的文档和Wireshark抓的包，整理了一下SSLSocket实验结果。

## 证书准备
证书是大家耳熟能详的Let's Encrypt签发的，我们可以用 [OHTTPS](https://ohttps.com/) 提供的服务来帮我们管理证书。<br>
使用邮箱注册，并将某个次级域名解析记录添加后，就可以生成证书（PEM类型）。<br>
生成的文件包括：私钥文件、证书文件、fullchain证书（包含证书和中间证书）。<br>
PEM格式的文件可以使用openssl来转成各种格式，而且我们同样可以用openssl来生成自签证书。

## 服务端
简单地配置了Spring Boot Web（https），证书按需要安装单证书（相当于自签证书）或fullchain证书。

## 原理及验证
![证书验证原理](https://github.com/lvv9/lvv9.github.io/blob/master/pic/image_2021-12-24_01-18-29.png?raw=true)

TLS协议第一个安全方面的验证，就是证书的验证
![SSL/TLS握手](https://github.com/lvv9/lvv9.github.io/blob/master/pic/image_2021-12-24_00-19-09.png?raw=true)
> The server sends the client a certificate or a certificate chain. A certificate chain typically begins with the server's public key certificate and ends with the certificate authority's root certificate.

就是说，证书的验证，是按证书链来验证的，如果服务端只安装单证书，一个默认的客户端实现会发生：
> javax.net.ssl.SSLHandshakeException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target

如果客户端同样安装自签证书，就可以验证通过：
```
    @Test
    public void testSelfSignedHandShake() throws Exception {
        KeyStore keyStore = KeyStore.getInstance(type);
        FileInputStream fileInputStream = new FileInputStream(ResourceUtils.getFile(resource));
//        InputStream inputStream = getClass().getResourceAsStream(path);
        keyStore.load(fileInputStream, pass.toCharArray());
        TrustManagerFactory trustManagerFactory =
                TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init(keyStore);
        SSLContext context = SSLContext.getInstance("TLSv1.2");
        context.init(new KeyManager[]{}, trustManagerFactory.getTrustManagers(), SecureRandom.getInstanceStrong());
        SSLSocket socket = (SSLSocket) context.getSocketFactory().createSocket("127.0.0.1", 8433);
        socket.startHandshake();
    }
```