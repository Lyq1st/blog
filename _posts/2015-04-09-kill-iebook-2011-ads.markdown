---
layout: post
title: "kill iebook 2011 ads"
date: 2015-04-09 19:03:40 +0800
comments: true
categories:
    - Tech
    - Security 
description: 去除iebook2011生成电子期刊的广告
keywords: 去除 iebook 2011 广告
redirect_from: /p/20150409/
---

## iebook2011电子期刊去广告

前些天一个同学在给公司做电子期刊，用了破解版怀疑中毒了，让我帮忙分析下程序有没有问题。丢到火绒里面看了下没有什么异常行为，iebook2011花money才有无广告版，为了放心的去广告，还是自己动手丰衣足食吧。
iebook2011生成的电子期刊在尾部会追加该文件的长度，擅自修改它的大小很容易crash。上windbg验证了下修改jmp条件跳过大小检测，后续会各种崩溃，反正只是为了去广告就另辟蹊径吧。
电子期刊只有一个exe，所有资源都在里面，用010editor打开二进制文件，看看它的广告吧。

```xml
<iebook>
	<global>
	.........
		<turnSoundEnabled>true</turnSoundEnabled>
		<Proload>0</Proload>
		<UseInitMc>false</UseInitMc>
		<readTimes>0</readTimes>
		<iebook_caption type="string">iebook xxxx</iebook_caption>
		<iebook_link>http://www.iebook.cn</iebook_link>
		<copyright>iebook xxxxwww.iebook.cn</copyright>
		<copylink>http://www.iebook.cn</copylink>
		<Mag_width>475</Mag_width>
		<Mag_height>650</Mag_height>
		<Window_width>1024</Window_width>
		<Window_height>768</Window_height>
		<bqzine>false</bqzine>
		<GlobalBackSound>_silent.mp3</GlobalBackSound>
		<FlipSound>sound1.mp3</FlipSound>
		<EditMode>false</EditMode>
		<ShowPageBorder>true</ShowPageBorder>
		<InitPage>0</InitPage>
		<TemplateMode>false</TemplateMode>
		<ToolsBarTop>true</ToolsBarTop>
		<ZineMemo></ZineMemo>
		<coverpng></coverpng>
		<bcoverpng></bcoverpng>
		<DateSumit>false</DateSumit>
		<DateGateway></DateGateway>
		<HotArea>0.1</HotArea>
		<script src=""/>
		<EmbedOCX>false</EmbedOCX>
		<keyword></keyword>
	</global>
	<bgsoud>
		<music name="iebook.mp3" file="iebook.mp3" inpage=""/>
	</bgsoud>
	<content background="bg.jpg" defaultpage="default.png">
		<page lpage="" rpage="9B4A11DB-0D8E-4DD0-9245-8930F174FEB8.jpg" movie="" lhard="0" rhard="0" viewPageNum="false" viewFrame="false" mask="true" page-caption="" movie-caption="15.jpg" usecolor="false" bcolor="16777215">
		</page>
		<page lpage="5F4D9432-F1A9-4740-A22A-97F5B5C418E3.png" rpage="5F4D9432-F1A9-4740-A22A-97F5B5C418E3.png" movie="" lhard="0" rhard="0" viewPageNum="false" viewFrame="false" mask="true" page-caption="">
		</page>
	</content>
</iebook>
```

以上是exe中嵌入的一段xml配置文件，包含了很多信息，page、page中资源、page的配置、全局配置等等。去广告主要是干掉底部的链接广告和左上角的广告，以及最后一页整个广告页面。

<!-- more -->

##KILL ADS

剩下就是很多体力活了，猜测它的校验机制，这里直接给出结论，只验证的了文件的大小。当然最好要保证文件各个偏移结构不变，要不然容易自找麻烦。
首先去掉page页，方法很多例如填充一样多的空格、直接用`<!-- -->`在长度相同的前提下注释掉，另一种就是修改它的xml中节点的属性。修改属性会带来缺页，不够完美，全部置为`0x20`就能够完美去除了。广告`page`中包含特征串`0BCEB8E4`，类似于`guid`的一个值，windows 开发中大家都喜欢用来作为唯一标识。`iebook_caption`中就是左上角显示的文字，`<copyright>`是下面广告相关的，思路有了，不需要的东西就用空格清理掉吧，想替换的位置自行替换。下面贴出代码：

```c
static const char  page_string [] = "<page lpage=\"0BCEB8E4";
static const char  page_string_end [] = "</page>";
static const char  node_viewframe [] = "viewFrame=\"";
static const char  iebook_caption [] = "<iebook_caption type=\"string\">";
static const char  iebook_caption_end [] = "</iebook_caption>";
static const char  iebook_copyright [] = "<copyright>";
static const char  iebook_copyright_end [] = "</copyright>";
static const char  iebook_link [] = "<iebook_link>";
static const char  iebook_link_end [] = "</iebook_link>";
static const unsigned int page_string_len = sizeof(page_string) - 1;
static const unsigned int page_string_end_len = sizeof(page_string_end) - 1;
static const unsigned int node_viewframeLen = sizeof(node_viewframe) - 1;
static const char clear_value [] = "true";
static const unsigned int clear_value_len = sizeof(clear_value) - 1;
static const unsigned int iebook_caption_len = sizeof(iebook_caption) - 1;
static const unsigned int iebook_copyright_len = sizeof(iebook_copyright) - 1;
static const unsigned int iebook_link_len = sizeof(iebook_link) - 1;
static const byte clear_string [] = "\0\0\0\0";

inline bool  MultiByteToUnicode(const char* strMultiByte, size_t strMultiByteLen, std::wstring& strUnicode, UINT uCodePage)
{
	int nUnicodeCount = MultiByteToWideChar(uCodePage, 0, strMultiByte, (int) strMultiByteLen, NULL, 0);
	if (nUnicodeCount <= 0)
		return false;
	if ((int) strMultiByteLen == -1)
		strUnicode.resize(nUnicodeCount - 1);
	else
		strUnicode.resize(nUnicodeCount);
	return MultiByteToWideChar(uCodePage, 0, strMultiByte, (int) strMultiByteLen, &strUnicode[0], nUnicodeCount) >= 0;
}
inline bool BDMUnicodeToMultiByte(const wchar_t* strUnicode, size_t strUnicodeLen, std::string& strUTF8, UINT uCodePage)
{
	int nUTF8Count = ::WideCharToMultiByte(uCodePage, 0, strUnicode, (int) strUnicodeLen, NULL, 0, NULL, NULL);
	if (nUTF8Count <= 0)
		return false;
	if ((int) strUnicodeLen == -1)
		strUTF8.resize(nUTF8Count - 1);
	else
		strUTF8.resize(nUTF8Count);
	return ::WideCharToMultiByte(uCodePage, 0, strUnicode, (int) strUnicodeLen, &strUTF8[0], nUTF8Count, NULL, NULL) >= 0;
}
inline std::string WtoA(const std::wstring &input)
{
	std::string strDest;
	BDMUnicodeToMultiByte(input.c_str(), input.length(), strDest, CP_UTF8);
	return strDest;
}
char *memstr(char *haystack, char *needle, int size)
{
	char *p;
	char needlesize = strlen(needle);

	for (p = haystack; p <= (haystack - needlesize + size); p++)
	{
		if (memcmp(p, needle, needlesize) == 0)
			return p; /* found */
	}
	return NULL;
}
bool crackFile(std::string st_str_path,std::string title)
{
	HANDLE hFile = NULL;
	hFile = ::CreateFileA(st_str_path.c_str(),
		GENERIC_WRITE | GENERIC_READ,
		0,
		NULL,
		OPEN_EXISTING,
		FILE_ATTRIBUTE_NORMAL,
		NULL);

	if (INVALID_HANDLE_VALUE == hFile)
	{
		DWORD dwError = GetLastError();
		return false;
	}
	ULARGE_INTEGER liFileSize;
	liFileSize.QuadPart = 0;
	liFileSize.LowPart = ::GetFileSize(hFile, &liFileSize.HighPart);
	byte*  byteOriginBuffer = new byte[liFileSize.LowPart], *byteBuffer = byteOriginBuffer;
	DWORD dwReadSize = 0;
	BOOL bRet = ::ReadFile(hFile, byteBuffer, liFileSize.LowPart, &dwReadSize, NULL);

	if (bRet)
	{
		{
			char *str_pos1 = memstr((char*) byteBuffer, (char*) iebook_caption, liFileSize.LowPart);
			if (NULL != str_pos1)
			{
				char *str_pos2 = memstr((char*) str_pos1, (char*) iebook_caption_end, liFileSize.LowPart - ((int) str_pos1 - (int) byteBuffer));
				if (NULL != str_pos2)
				{
					unsigned int caption_len = (unsigned int) str_pos2 - (unsigned int) str_pos1 - iebook_caption_len;
					printf("caption_len:%d\r\n", caption_len);
					if ("" == title)
						memset(str_pos1 + iebook_caption_len, 0x20, caption_len);
					else
					{
						memset(str_pos1 + iebook_caption_len, 0x20, caption_len);
						//std::string writeString = WtoA(title);
						std::wstring writeString;
						MultiByteToUnicode(title.c_str(), title.size(), writeString, CP_ACP);
						title = WtoA(writeString);
						if ( title.size() <= caption_len)
							memcpy(str_pos1 + iebook_caption_len, title.c_str(),title.size());
						else
							memcpy(str_pos1 + iebook_caption_len, title.c_str(), caption_len);

					}

				}
			}
		}
		{
			char *str_pos1 = memstr((char*) byteBuffer, (char*) iebook_copyright, liFileSize.LowPart);
			if (NULL != str_pos1)
			{
				char *str_pos2 = memstr((char*) str_pos1, (char*) iebook_copyright_end, liFileSize.LowPart - ((int) str_pos1 - (int) byteBuffer));
				if (NULL != str_pos2)
				{
					unsigned int copyright_len = (unsigned int) str_pos2 - (unsigned int) str_pos1 - iebook_copyright_len;
					printf("copyright_len:%d\r\n", copyright_len);
					memset(str_pos1 + iebook_copyright_len, 0x20, copyright_len);

				}
			}
		}
		{
			char *str_pos1 = memstr((char*) byteBuffer, (char*) iebook_link, liFileSize.LowPart);
			if (NULL != str_pos1)
			{
				char *str_pos2 = memstr((char*) str_pos1, (char*) iebook_link_end, liFileSize.LowPart - ((int) str_pos1 - (int) byteBuffer));
				if (NULL != str_pos2)
				{
					unsigned int link_len = (unsigned int) str_pos2 - (unsigned int) str_pos1 - iebook_link_len;
					printf("link_len:%d\r\n", link_len);
					memset(str_pos1 + iebook_link_len, 0x20, link_len);
				}
			}
		}
		{
			char *str_pos1 = memstr((char*) byteBuffer, (char*) page_string, liFileSize.LowPart);
			if (NULL != str_pos1)
			{
				char *str_pos2 = memstr((char*) str_pos1, (char*) page_string_end, liFileSize.LowPart - ((int) str_pos1 - (int) byteBuffer));
				if (NULL != str_pos2)
				{
					unsigned int page_len = (unsigned int) str_pos2 - (unsigned int) str_pos1 + page_string_end_len;
					printf("page_len:%d\r\n", page_len);
					memset(str_pos1, 0x20, page_len);
				}
			}
		}
		CloseHandle(hFile);

		st_str_path += "_crack.exe";
		hFile = ::CreateFileA(st_str_path.c_str(),
			GENERIC_WRITE | GENERIC_READ,
			0,
			NULL,
			CREATE_ALWAYS,
			FILE_ATTRIBUTE_NORMAL,
			NULL);
		if (INVALID_HANDLE_VALUE == hFile)
		{
			DWORD dwError = GetLastError();
			goto END;
		}
		else
		{
			DWORD dwWirteSize = 0;
			BOOL bRet = ::WriteFile(hFile, byteBuffer, liFileSize.LowPart, &dwWirteSize, NULL);
			if (bRet && liFileSize.LowPart == dwWirteSize)
			{
				printf("crack succeed\r\n");
			}
			CloseHandle(hFile);
		}
END:
		if (NULL != byteOriginBuffer)
		{
			delete [] byteOriginBuffer;
			byteOriginBuffer = NULL;
		}
		return true;
	}
	return true;
}
int main(int argc, char * argv []){
	if (2 == argc)
	{
		crackFile(argv[1],"");
	}
	else if (3 == argc)
	{
		crackFile(argv[1], argv[2]);
	}
	return -1;
}

```
想要直接使用exe的，可以直接到github上下载[KillIeBookAds.exe][]，cmdline:

```bat
KillIeBookAds.exe filepath title
```

稍微按照朋友意思定制了一下，能够改变左上角的title，字符数有限制<11，这个写下来没什么太多技巧大都是字符串的处理和文本编码问题，代码见github [kill_iebook_ad][]


[kill_iebook_ad]: https://github.com/Lyq1st/kill_iebook_ad
[KillIeBookAds.exe]: https://github.com/Lyq1st/kill_iebook_ad/blob/master/compiled_bin/KillIeBookAds.exe

