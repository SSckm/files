## 简介

```
一般的web服务器都是https单向认证，双向认证大部分用于企业对接，信任对方身份。比如该网站的接口不是所有人都可以访问，只允许一个或者部分持有证书的人访问。就需要双向认证
```

## 环境准备

```
ubuntu 16.4
nginx或者OpenResty, 本文用的OpenResty 版本为：openresty-1.11.2.1
```

## 创建目录

```
1.修改/etc/ssl/openssl.cnf配置文件（如果不知道openssl配置文件可以查找一下，后者使用locate openssl.cnf命令）
2.找到[CA_default]标签
3.修改dir对应的工作目录，我的目录为 /home/用户名/ca
4.cd 到工作目录
```

### 生成CA证书

```
openssl genrsa -aes256 -out private/ca.pem 1024
openssl rsa -in private/ca.pem -out private/ca.key
openssl req -new -key private/ca.pem -out private/ca.csr
openssl x509 -req -days 365 -sha1 -signkey private/ca.pem -in private/ca.csr -out certs/ca.cer
```

## 生成服务端证书

```
用根证书签发server端证书
openssl genrsa -aes256 -out private/server.pem 1024
openssl rsa -in private/server.pem -out private/server.key
openssl req -new -key private/server.pem -out private/server.csr
openssl x509 -req -days 365 -sha1 -CA certs/ca.cer -CAkey private/ca.pem -CAserial ca.srl -in private/server.csr -out certs/server.cer
```

## 生成客户端证书

```
openssl genrsa -aes256 -out private/client.pem 1024
openssl rsa -in private/client.pem -out private/client.key
openssl req -new -key private/client.pem -out private/client.csr
openssl x509 -req -days 365 -sha1 -CA certs/ca.cer -CAkey private/ca.pem -CAserial ca.srl -in private/client.csr -out certs/client.cer
```

## 导出证书

```
openssl pkcs12 -export -clcerts -inkey private/client.pem -in certs/client.cer -out certs/client.p12
openssl pkcs12 -export -clcerts -inkey private/server.pem -in certs/server.cer -out certs/server.p12
```

## 安装证书

```
1.安装CA证书，在谷歌浏览器中点击设置->高级选项->证书管理->授权中心，点击导入，然后选择生成的ca.cer证书文件。
2.安装客户端证书，在谷歌浏览器中点击设置->高级选项->证书管理->您的证书 点击导入，选择生成的client.p12证书文件
```

![](http://img.soaer.com/blog/11.png)

![](http://img.soaer.com/blog/13.png)



## 需要注意的地方

```
1.在生成客户端证书的时候在填写CN的时候不要和生成CA证书和服务端证书一样。测试好多次如果subj和CA，server一样的话浏览器一直是400错误。
```

## 配置nginx启动

```
server {
        listen       443 ssl;
        server_name  www.soaer.com;
        ssl on;
        ssl_certificate      /home/sunny/ca/certs/server.cer;
        ssl_certificate_key  /home/sunny/ca/private/server.key;
        ssl_client_certificate /home/sunny/ca/certs/ca.cer;
        ssl_verify_client on;
        ssl_prefer_server_ciphers on;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         HIGH:!aNULL:!MD5;

        location / {
            root   html;
            index  index.html index.htm;
        }
}
```

## 测试

```
浏览器中输入https://www.soaer.com, 如图所示,需要你选择发送给服务端的证书，此时选择client.p12证书文件
```

![](http://img.soaer.com/blog/10.png)

## 结果（成功）

![](http://img.soaer.com/blog/12.png)

