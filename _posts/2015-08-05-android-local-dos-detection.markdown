---
layout: post
title: "Android拒绝服务漏洞检测"
date: 2015-08-05 11:01:24 +0800
comments: true
categories: 
    - Tech
    - Security
description: Android 拒绝 服务 漏洞 检测
keywords: Android 拒绝 服务 漏洞 检测
redirect_from: /p/20150805/
---


###漏洞类型

目前发现的几类拒绝服务攻击：

- 通用型未定义类异常
- 空Action异常
- 数组越界异常
- 类型强转异常
- 访问不存在组件异常
- 其他异常

<!-- more -->
###漏洞介绍
#### 1.通用型未定义类异常

android4.4`/frameworks/base/core/java/android/os/Parcel.java readValue` 的实现代码，读取所有数据都会调用该函数:
{% highlight java lineanchors %}
1987    /**
1988     * Read a typed object from a parcel.  The given class loader will be
1989     * used to load any enclosed Parcelables.  If it is null, the default class
1990     * loader will be used.
1991     */
1992    public final Object readValue(ClassLoader loader) {
1993        int type = readInt();
1994
1995        switch (type) {
...
2055
2056        case VAL_BYTE:
2057            return readByte();
2058
2059        case VAL_SERIALIZABLE:
2060            return readSerializable();
...
2073
2074        default:
2075            int off = dataPosition() - 4;
2076            throw new RuntimeException(
2077                "Parcel " + this + ": Unmarshalling unknown type code " + type + " at offset " + off);
2078        }
2079    }
{% endhighlight %}

在解析`VAL_SERIALIZABLE`类型的数据则会调用`readSerializable`,
android4.4`/frameworks/base/core/java/android/os/Parcel.java readSerializable()` 的实现代码:

{% highlight java lineanchors %}
2196    public final Serializable readSerializable() {
2197        String name = readString();
2198        if (name == null) {
2199            // For some reason we were unable to read the name of the Serializable (either there
2200            // is nothing left in the Parcel to read, or the next value wasn't a String), so
2201            // return null, which indicates that the name wasn't found in the parcel.
2202            return null;
2203        }
2204
2205        byte[] serializedData = createByteArray();
2206        ByteArrayInputStream bais = new ByteArrayInputStream(serializedData);
2207        try {
2208            ObjectInputStream ois = new ObjectInputStream(bais);
2209            return (Serializable) ois.readObject();
2210        } catch (IOException ioe) {
2211            throw new RuntimeException("Parcelable encountered " +
2212                "IOException reading a Serializable object (name = " + name +
2213                ")", ioe);
2214        } catch (ClassNotFoundException cnfe) {
2215            throw new RuntimeException("Parcelable encountered" +
2216                "ClassNotFoundException reading a Serializable object (name = "
2217                + name + ")", cnfe);
2218        }
2219    }
{% endhighlight %}

Serializable未定义或者无法找到的类会抛出异常，而所有的getXXXExtra的实现都会调用readSerializable。

```java
Intent localIntent = new Intent();
localIntent.setComponent(new ComponentName(packageName, className));
localIntent.putExtra("serializable_object", new MalformedObject());

```
构造如上一个intent发送给解析extra数据的组件，APP组件会因为找不到`MalformedObject` class，抛出异常。

#### 2.空Action异常

这种异常通常是由于编码的不规范导致，没有对返回值做判空冗错处理，如：


```java
Intent intent = getIntent();
	if (intent.getAction().equals("android.intent.action.refuse.nullaction")) {
	    Log.d("TAG", "Test for Android Refuse Service Bug");
	}
```

`intent.getAction()`调用的返回值为空引发异常。

#### 3.数组越界异常

这类异常也是属于编码不规范导致，在使用数组类型的下标索引时，超出数组范围，如：


```java
ArrayList<Integer> intArray = intent.getIntegerArrayListExtra("interger_list");
	 	if (intArray != null) {
	 	    for (int i = 0; i < 100; i++) {
	 	        intArray.get(i);
	 	        Log.d("interger_list", "extra"+i);
	 	    }
	 	}
```

传入一个小于100的数组，遍历时会引起异常。
这类函数还有：

```java
		intent.getIntegerArrayListExtra("IntegerArrayList");
	 	intent.getStringArrayListExtra("StringArrayList");
	 	intent.getCharSequenceArrayListExtra("CharSequenceArrayList");
	 	intent.getBooleanArrayExtra("BooleanArray");
	 	intent.getByteArrayExtra("ByteArray");
	 	intent.getShortArrayExtra("ShortArray");
	 	intent.getCharArrayExtra("CharArray");
	 	intent.getIntArrayExtra("IntArray");
	 	intent.getLongArrayExtra("LongArray");
	 	intent.getFloatArrayExtra("FloatArray");
	 	intent.getDoubleArrayExtra("DoubleArray");
	 	intent.getStringArrayExtra("StringArray");
	 	intent.getCharSequenceArrayExtra("CharSequenceArray");
	 	intent.getParcelableArrayListExtra("ParcelableArrayList");
	 	intent.getParcelableArrayExtra("ParcelableArray");
```

在调用这些函数取值时都要注意边界的判断。

####4.类型强转异常

这类异常也是属于编码规范问题，没有对返回的值的类型做判断，直接强转引起异常，代码示例：

```java
	 	//Caused by: java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String
	 	String classCast = (String)intent.getSerializableExtra("serializable_object");
```
以上代码传入了一个`Integer`强转`String`引起崩溃。

#### 5.访问不存在组件异常

这类异常通常是开发者使用的第三方的组件，例如调用微信、新浪等分享的组件，由于系统内没有安装这些APP，在调用这些组件的时候会引起崩溃。

###漏洞检测方案

####问题分析
测试APK是否存在本地拒绝服务漏洞，只需要调用APK中的activity service broadcast组件，发送空Action、畸形类、数组数据，抓取日志匹配异常的关键字，就能判断是否存在拒绝服务问题。


####1.静态部分：
静态部分主要是针对数组越界异常的检测，分析出当前APK组件里面使用的数组类型的数据和key。分析出的key和数组类型作为动态fuzz的输入。

####2.动态部分：
大致的实现如下，当然细节处理起来是很繁琐的，例如多个模拟器的控制，fuzz任务的进度，模拟器down掉如何处理，结果如何存储等等。

- a. 启动模拟器
- b. 安装待检测APK
- c. 获取APK的各个组件
- d. 发送intent给个组件
- e. 抓取日志，结束后杀死进程
- f. 记录结果卸载APP

####3.Agent:
通过直接调用 am命令是无法发送复杂的数据的，如数组、对象等，因此针对这些fuzz类型要有一个装到android系统上的Agent作为桥梁，然后就可以随心所欲的构造intent了。

#### 小结

这个自动化的fuzz实现起来原理是比较简单的，但是中间的冗错处理非常的繁琐。

在模拟器的选择上还是要用原生的模拟器，相对来说稳定，问题也很明显：慢。当然在动态fuzz的过程中，也可以做些sql 注入、文件目录遍历问题的检测，更好的利用资源，毕竟安装卸载APK也是耗时的。