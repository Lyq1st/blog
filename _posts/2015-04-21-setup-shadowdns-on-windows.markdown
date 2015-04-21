---
layout: post
title: "Setup ShadowDNS on windows"
date: 2015-04-21 14:53:05 +0800
comments: true
categories: 
    - Conf
description: 在Windows上配置shadowDNS 
keywords: windows 配置 shadowDNS ssdns 
redirect_from: /p/20150421/
---

之前为了对抗DNS污染一直使用Dnsforwarder，公开的一些SecDNS不稳定，于是更换了方案，使用ssdns。

假定你已经有一个shadowsocks server，利用ssserver在Windows上配置一个dns forward绕过一些DNS干扰。

<!-- more -->
###基本环境状况


OS：Windows 7 x64 SP1   <br/> Python:　2.7 32bit
其他环境也可参考以下配置。

###配置m2crypto

下载 [OpenSSL for software developers][],这里ptyhon版本选择了32bit，因此openssl也选择32bit。安装完毕必须将openssl下的`bin、include`等文件拷贝到`C:\pkg`下[[?][]]，没有的话需要建该目录。
[OpenSSL for software developers]: http://slproweb.com/products/Win32OpenSSL.html 
下载[swig][],这里使用了3.0.4，使用最新的3.0.5要报错`AttributeError: 'module' object has no attribute 'PKCS5_SALT_LEN'`[[why?][]]，然后将swig解压，并将路径加入到环境变量PATH下。
[?]: http://stackoverflow.com/questions/6436771/easy-install-m2crypto-failing-on-windows-platform
[why?]: https://github.com/M2Crypto/M2Crypto/issues/24
[swig]: http://sourceforge.net/projects/swig/

以管理员身份运行

```bat
pip install m2crypto
easy_install pip
pip install shadowdns
```
###使用shadowDNS

建立配置文件*.json，参考[shadowDNS][]

```json
{
    "server":"my_server_ip",
    "server_port":8388,
    "local_address": "127.0.0.1",
    "password":"mypassword",
    "method":"aes-256-cfb",
    "dns":"8.8.8.8"
}

```

最后运行 

```bat
ssdns -c "*.json's path"
```

为了方便可以写一个简单的VB脚本，然后把VB脚本加到启动项中，让ssdns开机后台静默运行。VB脚本如下：

```C
set oshell = WScript.Createobject ("WSCript.shell")
oshell.run"ssdns -c server.json",0
wscript.quit
```

[shadowDNS]: https://github.com/shadowsocks/ShadowDNS