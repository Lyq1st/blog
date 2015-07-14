---
layout: post
title: "APK Tools GUI open source"
date: 2015-07-14 15:59:18 +0800
comments: true
categories: 
    - Tech
    - Security
description: A GUI for some apk analysis tool open source
keywords: GUI APK tools GUI Open Source
redirect_from: /p/20150714/
---

### Tool List

All tools is open source except jad.exe, you can use the testkey or generate a key by yourself.

- [dex2jar]
- [openssl]
- `testkey`
- [7z]
- [aapt]
- [apktool]
- [jad]
- [AXMLPrinter2]
- [baksmali]
- [signapk]
- [smali]

[dex2jar]: https://github.com/pxb1988/dex2jar
[openssl]: https://www.openssl.org/source/
[7z]: http://www.7-zip.org/7z.html
[aapt]: https://developer.android.com/tools/building/index.html
[apktool]: http://ibotpeaches.github.io/Apktool/
[jad]: http://varaneckas.com/jad/
[AXMLPrinter2]: https://code.google.com/p/android4me/
[baksmali]: https://code.google.com/p/smali/
[signapk]: https://code.google.com/p/signapk/
[smali]: https://code.google.com/p/smali/



<!-- more -->

### Function List

- `Decompile *.APK file`
- `Rebuild *.APK file`
- `Sign *.APK file`
- `Decompile *.DEX file`
- `Rebuild *.DEX file`
- `*.jar and *.class convert to *.java`
- `Get APK certificate`
- `Format AndroidManifest.xml`
- `Export log`

### UI
![APK_TOOLS_UI](/images/2015/07/14/apk_tools_gui.png "APK_TOOLS_UI")
### Source Code

You can get this project's code from my [github][]

[github]: https://github.com/Lyq1st/APK-tools-GUI

### Environment

This GUI tool is running on windows platforms, you can build this tool by vs2012 on windows 7.

