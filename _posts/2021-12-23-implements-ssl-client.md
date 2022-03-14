# SSL客户端开发验证

之前对密码学有一定程度的研究，这次本着get your hand dirty的原则，打算造一造轮子——一个简单的SSL客户端。项目代码见 [这里](https://github.com/lvv9/tls) <br>
SSL/TLS最广泛的应用当然是HTTP，所以一开始拿OkHttp的源码研究了一下。不久后发现，OkHttp是比较上层的，高层逻辑比较多，翻来翻去，翻到这份 [JSSE参考指南](https://docs.oracle.com/javase/7/docs/technotes/guides/security/jsse/JSSERefGuide.html) 。<br>
JSSE参考指南包含API框架和相应的实现，但是底层实现部分相关的资料比较少，对着几份官方的文档和Wireshark抓的包，整理了一下SSLSocket实验结果。

## 证书准备
证书是大家耳熟能详的Let's Encrypt签发的，我们可以用 [OHTTPS](https://ohttps.com/) 提供的服务来帮我们管理证书。<br>
使用邮箱注册，并将某个次级域名解析记录添加后，就可以生成证书（PEM类型）。<br>
生成的文件包括：私钥文件、服务器证书、fullchain证书（包含服务器证书、中间证书和根证书）。<br>
PEM格式的文件可以使用openssl来转成各种格式，而且我们同样可以用openssl来生成自签证书。

## 服务端
简单地配置了Spring Boot Web（https），证书按需要安装。

## 原理及验证

![SSL/TLS握手](https://github.com/lvv9/lvv9.github.io/blob/master/pic/image_2021-12-24_00-19-09.png?raw=true)
在上图的TLS握手中可以看到，协议第一个安全方面的验证，就是证书的验证，具体算法可见 [数字签名算法](https://zh.wikipedia.org/wiki/%E6%95%B0%E5%AD%97%E7%AD%BE%E5%90%8D%E7%AE%97%E6%B3%95) [DSA](https://en.wikipedia.org/wiki/Digital_signature) <br>
基于安全上等方面上的考虑，签发证书，是从CA分级签发的
![信任链](https://github.com/lvv9/lvv9.github.io/blob/master/pic/image_2021-12-28_01-41-22.png?raw=true)
> The server sends the client a certificate or a certificate chain. A certificate chain typically begins with the server's public key certificate and ends with the certificate authority's root certificate.

就是说，证书的验证，是按证书链来验证的，如果服务端只安装单证书，一个默认的客户端实现会发生：
> javax.net.ssl.SSLHandshakeException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target

通常，证书上的原文，包括证书所有者的域名等信息外，还会包括其生成的公钥。这个公钥，与签发机构签名用的私钥，并不是同一密钥对。<br>
从上图也可以看到，自签的证书，证书中的公钥与签发机构的私钥才是密钥对。而非自签的证书中，公钥与证书签名对应的私钥不是密钥对。<br>
理论上，需要认证的证书公钥，可以由被认证方生成并发送到签发机构进行签名。而在这里使用的OHTTPS，是OHTTPS系统帮忙生成并下发的（包括私钥）。

如果客户端信任签发的服务器证书，可以验证通过：
```
    @Test
    public void testEndCertHandShake() throws Exception {
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
根证书在各种应用场景默认会被安装，在证书链中，服务端fullchain证书中删除根证书不会影响证书的验证。

如果客户端直接信任中间证书，服务端只加载安装服务器证书，握手过程中也可以验证通过：
```
    @Test
    public void testSelfSignedHandShake() throws Exception {
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        FileInputStream fileInputStream = new FileInputStream(ResourceUtils.getFile(intermediate));
        keyStore.load(null);
        CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
        keyStore.setCertificateEntry("intermediate", certificateFactory.generateCertificate(fileInputStream));
        TrustManagerFactory trustManagerFactory =
                TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init(keyStore);
        SSLContext context = SSLContext.getInstance("TLSv1.2");
        context.init(new KeyManager[]{}, trustManagerFactory.getTrustManagers(), SecureRandom.getInstanceStrong());
        SSLSocket socket = (SSLSocket) context.getSocketFactory().createSocket("127.0.0.1", 8433);
        socket.startHandshake();
    }
```

还值得一提的是，证书中的公钥，除了被用来验证下级证书的合法性外，还可以在通信过程中来加密对称密钥，实现密钥交换（RSA算法）。
> Server key exchange: The server sends the client a server key exchange message if the public key information from the Certificate is not sufficient for key exchange. For example, in cipher suites based on Diffie-Hellman (DH), this message contains the server's DH public key.<br>
> The client generates information used to create a key to use for symmetric encryption. For RSA, the client then encrypts this key information with the server's public key and sends it to the server. For cipher suites based on DH, this message contains the client's DH public key.

## nginx反向代理
```text
    server {
        listen  8443 ssl;
        ssl_certificate  /etc/nginx/fullchain.cer;
        ssl_certificate_key  /etc/nginx/cert.key;

        location / {
            proxy_pass  http://hadoop1:9870/;
        }
    }
```
```shell
docker run --name https -v /Users/lwq/Desktop/nginx/nginx.conf:/etc/nginx/nginx.conf -p 8443:8443 --network hadoop -d nginx
```