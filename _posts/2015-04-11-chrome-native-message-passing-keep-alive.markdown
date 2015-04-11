---
layout: post
title: "Chrome Native Message Passing Keep-alive"
date: 2015-04-11 12:02:14 +0800
comments: true
categories:
    - Tech
description: chrome extension与本地app通信以及实现长连接
keywords: Chrome Native Message Passing Keep-alive
redirect_from: /p/20150411/
---

### Native Message Passing
`chrome` 官方文档有关于`native message`的解释[native message][],目的是取代以前使用的`native plug-in`，降低开发者在本地的权限，增加安全性。

`native message`出现于`chrome29`，其大致原理是`chrome`根据注册表中注册的`native app`的配置文件，找到`native application`，建立管道，通过标准的输入输出与本地程序通信，根据目前的版本来看（小于`43`的版本），`chrome`只负责`extension`的部署，`native app`的配置则需要自己在客户端上部署。`chrome` 启动或者开新`TAB`页都会拉起`native app`（只有一个），`chrome`进程全部关闭时`native app`退出（这个跟自己的`app`实现逻辑有关）

[native message]: https://developer.chrome.com/extensions/nativeMessaging

<!-- more -->

`manifest.json`和注册表的配置参考[native message][]，里面有详细的描述。

---
###Chrome Extension端的写法

假定你的`json`配置文件内容如下：

```json
{
  "name": "com.yours_company.yours_application",
  "description": "your Application",
  "path": "C:\\Program Files\\My Application\\yours_chrome_native_messaging_host.exe",
  "type": "stdio",
  "allowed_origins": [
    "chrome-extension://xxxxxxxxxxxxxxxxxxxxxxx/"
  ]
}
```

由于涉及到一些敏感信息，不方便直接贴出，以下是处理过后的`background.js`（反正extension代码是明的，可参考下百度杀毒上网保护的扩展）：

```js
function sendNativeMsg(msg) 
{
	connectNativeApp();
	port.postMessage(msg);
}
function onDisconnected() {

    port = null;
}
function connectNativeApp() 
{
    var hostName = "com.yours_company.yours_application";

    if (null == port) 
    {
    	port = chrome.runtime.connectNative(hostName);     
        port.onMessage.addListener(onNativeMessage);
        port.onDisconnect.addListener(onDisconnected);
    }
    if (null == port) 
    {
    }
}

function Init()
{
    connectNativeApp();
};

Init();
```
以上就完成了连接 `native app`和发送`native message`，如果需要处理回应的消息，可以再增加下相应的处理函数` port.onMessage.addListener(your_handle_function_callback)`,实现`your_handle_function_callback(message)`就能实现双向通信了。

---
###Native Application 端的写法

从[native message][]描述中可以得知，读取管道就能获取从浏览器抛过来的数据，至于协议格式，`chrome`并没有做详尽的说明。这里给出一个分析的结论：前四个`chars`表示抛过来的消息长度，后面紧跟的该长度大小的消息，命令行则是`chrome-extension://xxxxxxxxxxxxxxxxxxxxxxx/`这种形式。这里要注意前四个`chars`的处理方式，之前在stackoverflow上看到有这个处理，回答者就是处理这个头部长度时有问题，在高位不为空的情况下获取长度错误。

至于实现`Keep-live`则是让程序不退出，节约程序启动的资源使用，这里使用了一个`while`循环，代码如下：

```c
int APIENTRY _tWinMain(HINSTANCE hInstance,
					   HINSTANCE hPrevInstance,
					   LPTSTR    lpCmdLine,
					   int       nCmdShow)
{
	std::wstring inp;
	std::wstring strCmdLine = lpCmdLine;
	byte lenArr[4] = {0};
	// Check if this is being called as a native messaging host from chrome
	if (strCmdLine.find(L"chrome-extension://") != std::wstring::npos)
	{
		do 
		{
			if (lpCmdLine != NULL) 
			{
				std::wstring recv_message;
				unsigned int c, i = 0, t = 0;
				// Reset inp
				inp = L"";
				// Sum the first 4 chars from stdin (the length of the message passed).
				for (i = 0; i <= 3; i++) 
				{
					char inchar = getchar();
					lenArr[i] = inchar;
				}
				t = *((int*)lenArr);
				if( EOF == t )
				{
					return 0;
				}
				// Loop getchar to pull in the message until we reach the total
				//  length provided.
				for (i=0; i < t; i++)
				{
					c = getchar();
					inp += c;
				}
			}
			
		} while (true);
	}
	return 0;
}
```

后续时间空了会写一下这个通讯机制的完整`demo`，放到github上分享一下。
