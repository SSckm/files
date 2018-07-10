## 简介

```
由于最近在搞自己的个人技术网站，而收费的SSL证书对于我们个人站长来说又太贵，所以就申请了一下Let's Encrypt的免费证书。下面就线上申请和本地申请两种模式讲解一下生成证书的步骤以及方式
```

## 在线申请

```
打开网页：https://zerossl.com/
```

### 第一步

```
点击 ONLINE TOOLS
```

![](https://img.soaer.com/blog/22.png)

### 第二步

```
点击：START
```

![](https://img.soaer.com/blog/23.png)

###第三步

```
填写自己的邮箱和域名，注意域名不支持通配符。比如不支持*aaa.com,但可以支持多个。邮箱是选填。
最下面的两个要选择,点击 NEXT
```

![](https://img.soaer.com/blog/24.png)

### 第四步

```
这里我选择的是NO，可根据自己选择
```

![](https://img.soaer.com/blog/25.png)

###第五步

```
点击下载文件，下载生成好的文件。点击NEXT
```

![](https://img.soaer.com/blog/26.png)

### 第六步

```
下载生成的文件，点击NEXT
```

![](https://img.soaer.com/blog/27.png)

### 第七步

```
这个步骤是验证网站，点击下载文件，然后把文件放到根目录的 .well-known/acme-challenge/这个文件夹下面，这两个文件夹需要自己手动创建。而且这个文件每次生成的都不一样
```

![](https://img.soaer.com/blog/28.png)

### 第八步

```
第七步验证完网站并成功以后，就会生成两个文件，都是.txt结尾的文件。一个是domain.key.txt, 一个是domain.crt.txt, 需要改下后缀名，分别文.key和.crt.然后配置到web服务器就可以了。至此成功申请到免费的SSL证书。nginx配置不在此说了。
```



##本地生成

### 环境介绍

```
Ubuntu 16.04
```

###GitHub地址

```
https://github.com/do-know/Crypt-LE
克隆到本地
```

### 安装依赖组件

```
sudo apt-get install make gcc libssl-dev liblocal-lib-perl cpanminus
```

### 安装Crypt::LE

```
cpanm Crypt::LE
cpan -i Crypt::LE
perl Makefile.PL
make
make test
make install
```

## 命令生成证书

注意：在执行命令的时候如果不加上--live的话所有的命令都相当于在测试系统中生成的，浏览器是不信任测试系统的证书的，所以在生成正式环境的证书的时候需要加上--live选项

```
le.pl --key account.key --email "my@email.address" --csr domain.csr --csr-key domain.key --crt domain.crt --domains  "www.domain.ext,domain.ext" --generate-missing --live
```

说明：generate-missing的意思是你丢掉了所有的文件，包括私钥证书请求文件等。会重新生成

执行完命令后会有一下的日志输出，此时需要验证你的网站了。在网站的根目录创建：.well-known/acme-challenge/文件夹，然后创建日志提示的文件的名称，在这里文件名称为：G8mjTsaU1ks6yS5v7uEKGUQXHHPqj0rbu2vvxHLlktc， 在这个文件里面输入：G8mjTsaU1ks6yS5v7uEKGUQXHHPqj0rbu2vvxHLlktc.GFGRhimi-_yO_XP1LNO7sa3rb1BW752yGUB0PySF0nc内容，然后点击enter.

```
2018/01/29 11:15:32 [ ZeroSSL Crypt::LE client v0.29 started. ]
2018/01/29 11:15:32 Generating a new account key
2018/01/29 11:15:32 Saving generated account key into account.key
2018/01/29 11:15:32 Generating a new CSR for domains www.soaer.com
2018/01/29 11:15:32 New CSR will be based on a generated key
2018/01/29 11:15:34 Saving a new CSR into domain.csr
2018/01/29 11:15:34 Saving a new CSR key into domain.key
2018/01/29 11:15:35 Registering the account key
2018/01/29 11:15:36 The key has been successfully registered. ID: 5461161
2018/01/29 11:15:36 Make sure to check TOS at https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf
2018/01/29 11:15:36 Current contact details: liuzhenx@hotmail.com
Challenge for www.soaer.com requires:
A file 'G8mjTsaU1ks6yS5v7uEKGUQXHHPqj0rbu2vvxHLlktc' in '/.well-known/acme-challenge/' with the text: G8mjTsaU1ks6yS5v7uEKGUQXHHPqj0rbu2vvxHLlktc.GFGRhimi-_yO_XP1LNO7sa3rb1BW752yGUB0PySF0nc
When done, press <Enter>
```

OK，还是本地生成的快一点。