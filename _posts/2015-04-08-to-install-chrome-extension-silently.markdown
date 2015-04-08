---
layout: post
title: "To Install Chrome Extension Silently"
date: 2015-04-08 15:29:47 +0800
comments: true
categories: 
    - Tech
    - Security
description: 静默安装chrome extension
keywords: 静默 安装 chrome extension To Install Chrome Extension Silently
redirect_from: /p/20150408/
---

## 技术背景

chrome 用户量现在越来越多，很多产品做chrome扩展都想能够降低用户思考成本，一键安装。chrome extension store 在国内由于xx原因访问困难，即便是用户知道扩展链接，也难以自己安装。因此很多产品也在尝试静默安装扩展，看起来有点流氓
但是技术上不能说这是坏的，看你如何利用，目的是啥。

在`chrome 36`及其以前的版本`迅雷`、`alipay安全控件`都是支持静默安装的，原理都是修改用户的`preferences`配置文件。这些版本的chrome没有在`preferences `中严格校验配置是否有篡改，因此能够通过修改配置文件给用户安装`chrome` 扩展。

新版本的`chrome`对`preferences`做了严格的校验，如下是一个扩展的配置,配置文件名称不再使用`preferences`，而是使用`Secure Preferences`，在后续的有校验的版本中，这种暴力写入的方式就阵亡了，以下是一个extension安装过后的配置格式：

``` json
 "ahfgeienlihckogmohjhadlkjgocpleb": {
    "active_permissions": {
       "api": [ "management", "webstorePrivate" ],
       "manifest_permissions": [  ]
    },
    "app_launcher_ordinal": "t",
    "commands": {

    },
    "content_settings": [  ],
    "creation_flags": 1,
    "ephemeral_app": false,
    "events": [  ],
    "from_bookmark": false,
    "from_webstore": false,
    "incognito_content_settings": [  ],
    "incognito_preferences": {

    },
    "install_time": "13059811640710351",
    "location": 5,
    "manifest": {
       "app": {
          "launch": {
             "web_url": "https://chrome.google.com/webstore"
          },
          "urls": [ "https://chrome.google.com/webstore" ]
       },
       "description": "Chrome Web Store",
       "icons": {
          "128": "webstore_icon_128.png",
          "16": "webstore_icon_16.png"
       },
       "key": "MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCtl3tO0osjuzRsf6xtD2SKxPlTfuoy7AWoObysitBPvH5fE1NaAA1/2JkPWkVDhdLBWLaIBPYeXbzlHp3y4Vv/4XG+aN5qFE3z+1RU/NqkzVYHtIpVScf3DjTYtKVL66mzVGijSoAIwbFCC3LpGdaoe6Q1rSRDp76wR6jjFzsYwQIDAQAB",
       "name": "网上应用店",
       "permissions": [ "webstorePrivate", "management" ],
       "version": "0.2"
    },
    "page_ordinal": "n",
    "path": "C:\\Program Files\\Google\\Chrome\\Application\\37.0.2062.94\\resources\\web_store",
    "preferences": {

    },
    "regular_only_preferences": {

    },
    "was_installed_by_default": false,
    "was_installed_by_oem": false
 }
```
<!-- more -->
扩展的ID是`ahfgeienlihckogmohjhadlkjgocpleb`，在protection段中会有一串`256bit`的校验值，一旦发现配置被篡改，会在chrome settings页面提示配置被篡改，并让篡改的项目不生效。如下是一个Json结构的校验串:

``` json
"protection": {
	      "macs": {
	         "extensions": {
	            "known_disabled": "2A4FD778AC6D4A43DFCC2EF9F227F49AD086B5885B0AF8B8F111DCFE13FA9362",
	            "settings": {
	               "ahfgeienlihckogmohjhadlkjgocpleb": "F43A5159C5858BAD5923DB0F4A9BB1BA5724A9ADABAB278733FC1C53BD4AEF1F"
	            }
	         }
	   }
   }
```
多了这层校验目前来看所有厂商都无法进行静默安装扩展了，感谢chrome屏蔽掉了一些'盲流'、'祸害'....

奇虎360的产品同样有chrome上的extension，策略比较简单，直接通过传参数，打开chrome extension store 自己extension的链接，让用户安装，这样子实现成本比较低，但是受某墙影响，打不开extension的链接很正常。

还有一种静默安装chrome extension的方法是修改注册表，GitHub上有一篇挺详细的描述的文章 [Other Deployment Options][],这种静默安装扩展，用户卸载之后会加入黑名单，不再静默安装，同样要是依赖于网络都要受某墙影响，并且扩展不会自动启用，安装完毕后，chrome会有提示让用户启用，这不够'流氓'。

[Other Deployment Options]: https://github.com/lb-crx/doc/blob/master/source/1-guide/OtherDeploymentOptions.rst
---
## chrome extension preferences 校验机制


要分析校验机制最快的办法就是扒chrome源码部分了，这部分的是现在 `preferences`中。
校验码的计算的关键逻辑实现在`pref_hash_calculator.cc`中，代码片段如下。

``` c++
PrefHashCalculator::PrefHashCalculator(
    const std::string& seed,
    const std::string& device_id,
    const GetLegacyDeviceIdCallback& get_legacy_device_id_callback)
    : seed_(seed),
      device_id_(GenerateDeviceIdLikePrefMetricsServiceDid(device_id)),
      raw_device_id_(device_id),
      get_legacy_device_id_callback_(get_legacy_device_id_callback) {}

PrefHashCalculator::~PrefHashCalculator() {}

std::string GetMessage(const std::string& device_id,
                       const std::string& path,
                       const std::string& value_as_string) {
  std::string message;
  message.reserve(device_id.size() + path.size() + value_as_string.size());
  message.append(device_id);
  message.append(path);
  message.append(value_as_string);
  return message;
}

PrefHashCalculator::ValidationResult PrefHashCalculator::Validate(
    const std::string& path,
    const base::Value* value,
    const std::string& digest_string) const {
  const std::string value_as_string(ValueAsString(value));
  if (VerifyDigestString(seed_, GetMessage(device_id_, path, value_as_string),
                         digest_string)) {
    return VALID;
  }
  if (VerifyDigestString(seed_,
                         GetMessage(RetrieveLegacyDeviceId(), path,
                                    value_as_string),
                         digest_string)) {
    return VALID_SECURE_LEGACY;
  }
  if (VerifyDigestString(seed_, value_as_string, digest_string))
    return VALID_WEAK_LEGACY;
  return INVALID;
}
```
大致逻辑是`hash device_id + path + value_as_string`, `path` 与扩展ID相关，`value_as_string`与配置项相关。下面模拟chrome每一个计算的步骤，能够给出一个合法的校验值，从而绕过chrome对扩展配置检查，符合校验值。

---
## 构造配置合法校验值

- 计算`device_id`

device_id使用了开源的[rlz][],在chrome源码的第三方库中，编译依赖于base过于庞大，通过修改rlz测试程序，以Dll形式导出获取device_id的接口，编译好的dll我会提交到github上。调用`_GetMachineId(in, MAX_MACHINE_ID_LEN))`获取`Raw device ID`

[rzl]: https://github.com/rogerta/rlz

- 计算`path`

`path`的构造方式为`extensions.settings.extension_ID`,`extension_ID`是extension的ID拼接字串作为`path`。

-获取 `value_as_string`

`value_as_string` 为extension的配置文件，可以通过安装扩展后，从preferences中获取该扩展实际的配置，形式如上一节中的`ahfgeienlihckogmohjhadlkjgocpleb`配置。

- 构造`message`

从`GetMessage`中不难看出，构造的过程只是做了简单的字符串的拼接，就是以上的三个值进行`device_id + path + value_as_string`,生成`raw message`

- 获取`hash seed`

`hash seed`用于计算Hash值，它存在于`resources.pak`中，该文件类似于`*.zip`结构。文件头部格式如下:

```c
typedef struct _DATAHEAdER{
	uint32 versions;
	uint32 entries;//total source count
	uint8 encoding;

} DATAHEAdER,*LPDATAHEAdER;
```
头部结束后紧跟资源内容，`resource_id` 和该资源的偏移 `file_offset`其结构如下：

```c
typedef struct _DATAPACKENTRY{
	uint16 resource_id;
	uint32 file_offset;
}DATAPACKENTRY,*LPDATAPACKENTRY;
```
枚举`entries`个资源结构，就可以遍历整个`resources.pak`的资源，`seed` 在不同的版本的`resource_id`不一定相同，需要定制该值，也可以根据经验值（目前所有版本的`seed`长度为64，且唯一），来寻找`seed`

- 计算`Hash`值

`raw message` 构造完成，通过[HMAC][]来计算`hash`（chrome中自带的hmac依赖过多不建议采用）。计算时选择`SHA256_DIGEST_SIZE)`，并指定seed，关键代码如下：

```c
std::string message;
	message.append( device_id );
	message.append( path );
	message.append( plaint );

	unsigned char mac[SHA512_DIGEST_SIZE];
	hmac_sha256((const unsigned char *)pre_hash_seed_bin.c_str(), pre_hash_seed_bin.length(), (unsigned char *) message.c_str(),
		message.length(), mac, SHA256_DIGEST_SIZE);
```

[HMAC]: https://github.com/ogay/hmac/commits/master

- 写入secure preferences

计算出合法的`hash`就可以写入配置了，使用json库来解析secure preferences文件，模拟chrome写入的配置和`hash`值。关于chrome preferences中字段的意义可以参考chrome extension开发文档。

- 释放`crx`扩展文件

解压`*.crx`扩展文件，crx是`*zip`文件的变种，头部略有不同，处理一下使用`7z`解压到指定目录 `extension_id\\version`（扩展ID拼接上version版本号）