## 从S2-052补丁分析Xstream反序列化漏洞修复方案

### 概述

前段时间在审计代码时发现一处Xstream反序列化漏洞，这个漏洞并不是什么新鲜的东西。但在百度、谷歌翻了一遍，大多都是Xstream反序列化漏洞分析及利用的相关技术文章，始终没有找到关于修复方案的详细介绍或者示例，尤其是基于白名单类校验的修复方案。作为一个从事应用安全的人来说，发现漏洞只是一部分，如何修复漏洞也是我们要考虑的一部分。于是想起了由lgtm.com的安全研究员提交的Struts2 REST插件的XStream组件存在反序列化漏洞(S2-052),所以通过S2-052的补丁分析整理出了基于白名单的xStream.fromXML反序列化漏洞的修复方案。

### 漏洞原理

首先简单说下这个漏洞产生的原理，程序代码在使用Xstream的fromXML方法将xml数据反序列化为java对象时。当输入的xml数据可以被用户控制，那么攻击者可以通过构造恶意输入，让反序列化产生非预期的对象，在此过程中执行构造的任意代码。

漏洞代码示例如下：

``` java
  ......
  //读取对象输入流,并进行反序列化
  InputStream importStream = importFile.getInputStream();
  importXML = IOUtils.toString(importStream, "UTF-8");
  XStream xStream = new XStream();
  //调用fromXML进行反序列化
  importData = (Myclassname) xStream.fromXML(importXML);
  ......
```

### 代码审计

代码中直接利用请求输入的xml数据作为fromXML的参数使用，这里输入可能是输入流、文件、post参数等。并且没有设置允许反序列化类的白名单，即可确认存在反序列化漏洞。

### S2-052补丁分析

Apache Struts对于S2-052的修复补丁地址:

https://github.com/apache/struts/commit/19494718865f2fb7da5ea363de3822f87fbda264

主要的修复实现代码在XStreamHandler.java文件中，其中增加了新的XStream初始化函数createXStream(ActionInvocation invocation).初始化函数中主要是调用了XStream的addPermission方法，首先清除默认设置，然后进行自定义设置并将Collection、Array、NULL、primitive这些些基础类、时间类和自定义的类放在了允许反序列化类的白名单中。

### 修复方案

如果可以明确反序列化对象的类名，则可在反序列化时设置允许被反序列化类的白名单（推荐），
如果不能确定反序列化对象的类，则在反序列化时设置不允许被反序列化类的黑名单。具体实现方法如下:

#### 使用白名单校验方案(推荐)
通过Xstream的addPermission方法来实现白名单控制，示例代码如下:

``` java
XStream xstream = new XStream();
// 首先清除默认设置，然后进行自定义设置
xstream.addPermission(NoTypePermission.NONE);
// 添加一些基础的类型，如Array、NULL、primitive
xstream.addPermission(ArrayTypePermission.ARRAYS);
xstream.addPermission(NullPermission.NULL);
xstream.addPermission(PrimitiveTypePermission.PRIMITIVES);
// 添加自定义的类列表
stream.addPermission(new ExplicitTypePermission(new Class[]{Date.class}));
// 添加同一个package下的多个类型
xstream.allowTypesByWildcard(new String[] {Blog.class.getPackage().getName()+".*"});
```
可以根据业务需求设置白名单。在设置前一定要先清除默认设置，即addPermission(NoTypePermission.NONE)。在辛苦分析并测试方案后一位朋友说Xstream官方也有对白名单设置的代码示例。走了不少弯路，所以总结出来希望其他朋友可以看到。Xstream同时内置了很多类型，可以参考[Xstream官方示例](http://x-stream.github.io/security.html#example)

#### 使用黑名单校验方案
使用Xstream的denyPermission方法可以实现黑名单控制，但不推荐使用该方法。

### 安全建议

1. 业务需要使用反序列化操作时，尽量避免反序列化数据由外部输入，这样可避免被恶意用户控制
2. 更新org.codehaus.groovy等第三方依赖库版本,已公开的关于xstream反序列化漏洞利用方式是通过Groovy的漏洞CVE-2015-3253，只要Groovy版本在1.7.0至2.4.3之间都受影响。所以建议修复本漏洞的同时也更新groovy等依赖库的版本

### 参考文档
* [S2-052补丁链接](https://github.com/apache/struts/commit/19494718865f2fb7da5ea363de3822f87fbda264)
* [Xstream官方示例](http://x-stream.github.io/security.html#example)
* [S2-052漏洞分析](http://blog.nsfocus.net/struts2-s2-052-rest-plug-in-remote-code-execution-technical-analysis/)

