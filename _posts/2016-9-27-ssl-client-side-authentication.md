---
layout: post
title: SSL 客户端验证
date: 2016-9-27 04:00
tags: [SSL]
---

对于自己的个人网站，安全要求没有那么重要。如果想对网站做一个简单的身份验证，可以有两种方式：

1. 直接在 Nginx 中配置一个简单的 `auth_basic` 形式的用户名 & 密码；
2. 或者使用本文要说的 **SSL Client Side Authentication**；

注：由于 __Client Side Authentication__ 太长，下文会用 **CSA** 这个缩写来代替。

## 为什么用这个

两个原因：

1. 安全。这个不用说了，只有拥有证书的人才能被认证，就排除了密码被破解的可能性；
2. 方便。不用输入用户名密码什么的了啊；

## 配置过程

首先需要明确，CSA 的配置过程和网站的 **Server Side Authentication** 是完全独立的，两者没有什么关系。

### 环境准备

安装 openssl 工具。

```bash
brew install openssl
```

准备一个空目录。

```bash
mkdir certs
cd certs
```

### 创建根证书

```bash
# generate primary key
openssl genrsa -des3 -out ca.key 4096
# or if you don't want a password:
openssl genrsa -out ca.key 4096
# generate a cert
openssl req -new -x509 -days 365 -key ca.key -out ca.crt
```

生成 ca.crt 文件的时候会需要填写一些信息，看着填就好了，关系不大。我是这么填写的：

```bash
$ openssl req -new -x509 -days 365 -key ca.key -out ca.crt
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
------
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Beijing
Locality Name (eg, city) []:Beijing
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:Hongbo He
Common Name (e.g. server FQDN or YOUR name) []:Hongbo He
Email Address []:me@graycarl.me
```

然后生成的证书长这样：

 ![cert-1](/fs/17-08-03-600e4f7a.png)

这里创建的 ca.crt 是会放到服务端的，接下来我们创建安装在客户端的证书。

### 创建客户端证书

首先是创建证书请求：

```bash
openssl genrsa -out client.key 4096
openssl req -new -key client.key -out client.csr
```

这里同样会需要输入一些信息，看着输入就好。

然后使用 ca 证书进行签发：

```bash
openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out client.crt
```

这样就签发好了客户端证书 client.crt。

### 安装客户端证书

上一步产生的 `client.crt` 并不能直接安装，需要转成 PKCS 格式：

```bash
openssl pkcs12 -export -clcerts -in client.crt -inkey client.key -out client.p12
```

这里的 client.p12 在 MacOS 上就可以直接双击安装到系统的 KeyChain 里面。

### 部署到服务器

服务端只需要一个 ca.crt 文件即可。

```bash
scp sa.crt somebody@someserver:/etc/nginx/certs/client-auth-ca.crt
```

Nginx 中加入如下的配置：

```nginx
server {
    ...
    ssl_client_certificate /etc/nginx/certs/client-auth-ca.crt;
    ssl_verify_client optional;
    ...
}
```

其中 `ssl_verify_client` 的值可以是 `optional | on` 者两个：

1. 当选择 `on` 时会强制进行客户端认证，失败无法访问；
2. 当选择 `optional` 的时候，认证是可选的，是否认证成功可以从 `$ssl_client_verify` 变量得知。

比如，我们想要网站根目录是任意访问的，但是 `/admin` 路径下是需要认证才能访问的，就可以这么配置：

```nginx
server {
    ...
    ssl_client_certificate /etc/nginx/certs/client-auth-ca.crt;
    ssl_verify_client optional;
    ...

    location /admin {
        if ($ssl_client_verify != SUCCESS) {
            return 401;
        }
        proxy_pass http://localhost:5000;
    }
}
```

## 完成

当打开网站且本机装有对应的客户端证书时，就会出现请求证书认证的提示框：

![cert-2](/fs/17-08-03-704063d1.png)

点确定就可以通过认证啦。

## 参考

[ClientSide SSL](https://gist.github.com/mtigas/952344)

[Nginx SSL Module](http://nginx.org/en/docs/http/ngx_http_ssl_module.html)
