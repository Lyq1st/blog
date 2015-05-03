---
layout: post
title: "Use Chrome Extension Installer"
date: 2015-05-03 14:33:23 +0800
comments: true
categories: 
    - Tech
    - Security
description: Use Chrome Extension Installer 安装 chrome 扩展 静默 代码
keywords: Use Chrome Extension Installer code silently 安装 chrome 扩展 静默 代码
redirect_from: /p/20150503/
---

### ChromeExtInsaller

ChromeExtInsaller是静默安装的核心部分，目前静默安装功能需要配合一些手工操作，首先要获取到想要安装的扩展安装后在preferences 或者Secure preferences中生成的配置，在extinstaller.h权限配置、版本、ID等修改。

<!-- more -->

###rlz_id
该dll一直没有独立拆分，是在chrome 38上面的third party 中修改出来的，对外提供一个获取machine id 的接口，因此这里只有二进制，没有提供source code，需要的话要自行编译。

###native app 部署

如果需要native app 需要配置相应的host，会自动生成相应的*.json文件，部署注册表信息等。

###chrome Installer部署
chromeExtInstall.exe rlz_id.dll bdchromeExt.crx 部署在同一目录下，bdchromeExt.crx为百度杀毒的扩展，这里用来作为一个安装的例子。

###其他

如果需要了解其中详细的安装流程可以参考[To Install Chrome Extension Silently][],已经在github上提供了代码 [chromeExtInstaller][].

###TO DO

- 时间允许的话，后续会分析一下config的生成，取代当前需要手工安装扩展提取配信息的过程
- rlz lib能够合并进来

[To Install Chrome Extension Silently]: http://blog.droid-sec.com/2015/04/08/to-install-chrome-extension-silently/
[chromeExtInstaller]: https://github.com/Lyq1st/chromeExtInstaller