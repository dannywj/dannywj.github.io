---
title: HEXO+Github静态博客搭建小试牛刀
date: 2016-01-13 16:44:22
tags: develop
---
# 缘起
最近无意间看到了nodejs的火爆已经成为前后端通吃的一个技术了，之前也小研究过一些，今天想从另一个方面使用一下它的“功能”，玩一玩这个神奇的技术。

查了一些资料，github支持了用户发布一些静态资源在上面，用于自己的博客啊或者个人主页之类的。参考了一些前人的作品，发现了基于nodejs的一个框架 **HEXO** 于是开始了一些尝试，准备用此框架来生成一个自己的技术博客。

# 准备工作
本着简单实用，所有的操作都是在windows平台上来完成的。

## nodejs
*一句话带过*：官网下载exe安装包直接安装即可（可能需要配置环境变量）

## git
*一句话带过*：下载git包安装，有gitbash 即可（不是小乌龟客户端）

## npm
npm是nodejs的第三方包管理器，只需要配置好json文件，就可以下载制定版本的nodejs第三方库，非常方便。下面的HEXO就是通过npm的方式得到的。

## hexo
*一句话带过*：官网：[https://hexo.io/](http://hexo.io/)有安装和配置简介。
## hexo插件
*一句话带过*：由于hexo需要使用部署，项目生成，服务器模拟等功能，需要安装一些插件来辅助操作。插件安装在了项目目录的：node_modules文件夹下。

## 主题
选择hexo的一个重要的原因上github上有很多基于此的主题，使自己的网站看起来与众不同又可以自由定制。
## github pages
github账号就不用说了，肯定得有。但是在github上托管的项目不是每一个都能成为博客的。必须要建立一个特定名字的项目
**github用户名.github.io** 这样的格式，在项目配置标签页里有博客设置的界面。
其中，官方也提供了一些内置的模板供选择。


# 环境配置&调试
```
    $ npm install hexo-cli -g
    $ hexo init blog
    $ cd blog
    $ npm install
    $ hexo server

```
首先就是使用默认的模板+hello world默认博文来演示。遇到的一个坑爹的困难就是hexo初始化及生成完毕以后，在本地无法运行。
后来查到的原因是hexo server默认使用4000端口，而自己电脑的4000端口被福昕阅读器的一个进程给占用了。。！而且结束不掉！
索性，根据文档，换个端口呗

```
$ hexo s -p 5000 #指定使用5000端口来演示
```
http://localhost:5000 Success!
非常激动，调试成功。

网站基本的配置都在根目录下：_config.yml 文件里
yml的文件格式冒号后面得空一个格。。好吧。

# 主题配置&修改
主题的配置只需要将喜欢的主题clone到根目录下面的theme文件夹下，然后修改一下配置文件的theme属性，指定成clone的文件夹名称，重新生成，server一下即可。

关于这个主题，主题内的配置需要在主题文件夹里面的_config.yml文件中修改。如修改头像，分类，个人描述，分享链接等等。

另外，如果主题包里有多个语言，需要在网站的总配置文件里配置语言，即language目录的文件名字 如：zh-CN

# HEXO常用命令
```
$ hexo new "新建文章" #新建文章
$ hexo g #生成
$ hexo s -p xxxx #本地部署
$ hexo d #发布到github
```
# github部署
部署这块也踩了不少坑，首先是配置，网上的教程可能过时了，还是得以官网的说明为准：
有以下几个配置项：
```
deploy:
  type: git
  repo: <git path>
  #branch: master 可选
  message: add blog #提交注释
```
其中的repo的格式弄了半天 应该是：

```
https://<username>:<password>@github.com/<username>/<repository-name>

```

这样的
哎~

# 小插曲——instagram影集集成
主题里集成了一个引用instagram的模块，自己也有照片在instagram上，折腾了一下。
遇到几个问题：
1.漏引了jquery，后来自己加上了。
2.需要申请instagram的开发者资格，注册应用，获取clientid
3.sandbox应用默认有协议不允许访问资源，需要手工取消。
4.由于没有正式申请应用，可能没有永久使用照片接口的权限。因此手工生成了access token，将地址写死在了js文件里。。
5.需要翻墙

```
OAUTH授权信息：
Successfully registered 'DannyBlog'
DannyBlog
Client Info
Client ID 	4f5494bc5c614f188fa657427ae78b48
Client Secret 	f0957e6831f04bb6a7dxxxxxxx
Website URL 	http://dannywj.github.io/
Redirect URI 	http://dannywj.github.io/
Support Email 	dannywjwj@gmail.com

手动换取token：
https://instagram.com/oauth/authorize/?client_id=4f5494bc5c614f188fa657427ae78b48&redirect_uri=http://dannywj.github.io/&response_type=token

access_token=2003826149.4f5494b.06fd5c9c7844444da321f185d52xxxx

获取照片数据：
写死在主题包的js文件里
https://api.instagram.com/v1/users/2003826149/media/recent/?access_token=2003826149.4f5494b.06fd5c9c7844444da321f185d52xxxx
```


# 说点别的
经过一系列的折腾，终于成功部署上去了。虽然真正没有写几行代码，但是这个过程还是收获了很多的，熟悉了一些命令，进行了一些测试，在解决问题的时候需要进行一些思考，也许这个问题不仅仅是这个项目遇到的，可能是另外的问题，无疑锻炼了自己解决问题，搜索答案的能力。

而且，这个系统用来写博客，可以方便的跟踪每个版本的变化。

# 日常发布&环境重现
1. git clone项目
2. 切换分支到raw
3. 在raw中执行hexo -g
4. 根据报错内容，安装缺失的hexo组件
5. raw为发布分支，master为显示分支

npm install hexo --save （可能导致报错）

安装Hexo缺失插件（批量复制粘贴）
```
npm install hexo-generator-index --save
npm install hexo-generator-archive --save
npm install hexo-generator-category --save
npm install hexo-generator-tag --save
npm install hexo-deployer-git --save
npm install hexo-deployer-heroku --save
npm install hexo-deployer-rsync --save
npm install hexo-deployer-openshift --save

npm install hexo-server --save
npm install hexo-renderer-stylus@0.2 --save
npm install hexo-generator-feed@1 --save
npm install hexo-generator-sitemap@1 --save
npm install hexo-renderer-marked@0.2 --save
http://www.chinaz.com/web/2015/1016/458004.shtml
```

如果显示的是正常网页，则说明没有错误，但是如果是这样的：

<%- partial('_partial/head') %>
<%- partial('_partial/header') %>
<%- body %>
<% if (theme.sidebar && theme.sidebar !== 'bottom'){ %> <%- partial('_partial/sidebar') %> <% } %>
<%- partial('_partial/footer') %>
<%- partial('_partial/mobile-nav') %> <%- partial('_partial/after-footer') %>

这说明服务器解析错误，解决方案：
```
npm install hexo-renderer-ejs --save
npm install hexo-renderer-stylus --save
npm install hexo-renderer-marked --save
```

# 参考链接
[http://ijiaober.github.io/categories/hexo/](http://ijiaober.github.io/categories/hexo/)
[http://www.zhihu.com/question/24422335](http://www.zhihu.com/question/24422335)
[https://github.com/litten/hexo-theme-yilia](https://github.com/litten/hexo-theme-yilia)
[http://stackoverflow.com/questions/20871549/error-when-push-commits-with-github-fatal-could-not-read-username](http://stackoverflow.com/questions/20871549/error-when-push-commits-with-github-fatal-could-not-read-username)
http://www.chinaz.com/web/2015/1016/458004.shtml
[https://blog.csdn.net/u012366161/article/details/42059813](https://blog.csdn.net/u012366161/article/details/42059813)