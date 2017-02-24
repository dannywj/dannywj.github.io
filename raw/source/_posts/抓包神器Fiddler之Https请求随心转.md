---
title: 抓包神器Fiddler之Https请求随心转
tags: [develop,tools]
date: 2017-02-24 11:56:32
---

随着AppleStore对APP的审核越来越严格，客户端请求服务端API的方式大多数都变更为了https，在更安全的同时又引起了另外一个问题——**本地抓包开发调试的不便**。


一般来说，我们在开发API的时候，**本地环境基本都是不支持https的**（若要支持https则需要安装证书，比较麻烦），抓包神奇Fiddler由于拥有出色的功能，可以对请求进行拦截和处理，因此我在想能不能有一种办法，把所有的https请求自动转换成http呢，这样不就方便了吗！


前期曾经尝试过一种方法，那就是Fiddler里的AutoResponder选项卡里的EnableRules功能，主要是根据指定的规则来过滤https请求，然后手工改成http。

如下图所示：
![](http://images2015.cnblogs.com/blog/541749/201702/541749-20170224112250320-1590853713.png)

这种方法可以针对单个请求来实现转化，但是如果**APP的请求很多，每个API都要处理一遍**的话，难免会力不从心，能不能批量统一处理呢？

**答案是可以的！**
这需要借助++Fiddler的Customize Rules++功能来帮忙！
![](http://images2015.cnblogs.com/blog/541749/201702/541749-20170224112321788-2011819056.png)

这个功能是不随Fiddler的安装默认安装的，第一次使用会弹出提示安装的对话框，进行安装即可。

附插件安装地址：*http://fiddler2.com/r/?SYNTAXVIEWINSTALL*

这个插件的功能是针对抓取的请求，可用自定义脚本的方式来进行处理。

通过查阅资料，我们需要修改OnBeforeRequest方法，在里面添加“https转http”的逻辑。

代码如下：
~~~C#
// Handle HTTPS requests
if (oSession.isHTTPS)
{
    oSession.fullUrl = "http://" + oSession.hostname + oSession.PathAndQuery;
}
~~~
添加后点击保存按钮，**再次抓包，奇迹出现了！**

![](http://images2015.cnblogs.com/blog/541749/201702/541749-20170224112353882-151827977.png)

所有的https请求都被转换成http请求了！在本地可以愉快的调试API代码了~

参考资料：
https://groups.google.com/forum/#!topic/httpfiddler/TiR0fm8PLew
http://stackoverflow.com/questions/10040483/https-http-via-fiddler
