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
- 空指针访问异常
- 数组越界异常
- 类型强转异常
- 访问不存在组件异常
- 其他异常

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




