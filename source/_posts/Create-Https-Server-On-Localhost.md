---
title: 在本地创建https服务器
date: 2019-11-21 17:22:11
tags:
---

之前开发了一个需求，需要用到地理位置API(Geolocation) 。但基于个人隐私安全的考虑，一些涉及到个人隐私的API比如地理位置、摄像头、录音等都必须在安全源下才有效。所以需要在本地启动https服务，并且这个服务需要持有浏览器承认的有效证书。

浏览器请求https网站的时候，为了确认服务端是安全的，会在ssl握手的时候向请求证书，并且检查这个证书的有效性。浏览器都会内置一些权威的证书颁发机构，只要这个证书是这些颁发机构（CA）进行签名的，浏览器就承认其有效性。

向权威机构申请证书是收费的。如果只是本地开发的话，我们可以使用自签证书，然后让系统信任即可。

下面展示在Mac下的完整的步骤，其他系统下差别不到。

# 创建根证书颁发机构
首先创建一个项目目录
```shell
mkdir ~/study/https-server
cd ~/study/https-server
```

我需要用到`openssl`这个工具，如果没有安装的话先安装：
```shell
brew install openssl
```

## 生成私钥
```shell
openssl genrsa -des3 -out rootCA.key 2048
```
执行后输入密码，这里举例输入`123456`并回车

## 生成根颁发机构证书
`-key`选项传入上一步生成的私钥`rootCA.key`
```shell
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem
```
输入上一个步骤设置的密码`123456`，后面的都可以随便填

## 信任根证书
双击上一步生成的`rootCA.pem`，进入系统证书列表，双击证书，设置为始终信任
![](/images/ca-setting.png)

# 生成服务器证书

## 创建证书签名请求需要的配置`csr.cnf`
注意Common Name字段`CN`填写服务器域名，这里填写`localhost`
```shell
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C=CN
ST=Test
L=Test
O=End Point
OU=Testing Domain
emailAddress=123@qq.com
CN = localhost
```

## 生成私钥和证书签名请求文件(CSR)
`-config`输入刚刚创建的`csr.cnf`
```shell
openssl req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout server.key -config csr.cnf
```
前面的可以随便签，密码这里举例输入`abcde`

## 创建扩展配置文件`v3.ext`
注意`DNS.1`填写服务器要使用的域名，这里填写`localhost`
```shell
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
```

## 签名的服务器证书
`-in`传证书签名请求`server.csr`
`-CA`传根证书`rootCA.pem`
`-CAKey`传跟证书私钥
`-extfile`传刚刚的配置文件`v3.ext`
```shell
openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 500 -sha256 -extfile v3.ext
```
会车，输入证书密码`123456`后回车，生成证书文件`server.crt`，至此证书生成完毕。
我们使用`server.key`和`server.crt`创建https服务器

# 创建服务器

新建文件`index.js`
`options.key`使用服务器证书私钥`server.key`
`options.cert`使用服务器证书`server.crt`
```shell
const https = require('https');
const fs = require('fs');

const options = {
  key: fs.readFileSync(__dirname + '/server.key'),
  cert: fs.readFileSync(__dirname + '/server.crt')
};

https.createServer(options, (req, res) => {
  res.writeHead(200);
  res.end('hello world\n');
}).listen(443);
```
启动服务器
```shell
sudo node index.js
```

浏览器访问`https://localhost`
可以看到直接输出`hello world`，且浏览器没有任何安全警告
点击地址栏旁边的锁，可以看到证书是有效的
![](/images/safe-connect.png)

大功告成！