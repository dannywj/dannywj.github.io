---
title: WAMP工作环境配置
tags: develop
date: 2016-03-23 14:31:07
---

WAMP工作环境配置主要包括php环境搭建以及一些常用扩展的安装和配置。

# 安装php环境
版本5.4+
## 集成工具：XAMPP
安装完成后，如果出现403报错，需要修改权限配置：
```
找到\xampp\apache\conf\httpd.conf配置文件
解决方法：
找到Directory 目录然后进行复制权限；主要红色字

<Directory />
    Options FollowSymLinks
    AllowOverride None
    Order deny,allow
    Deny from allow
</Directory>
```



# 配置php扩展
## 预先检查
### VC9 or VC11
查看方法：
phpinfo中
搜索：Compiler
示例：MSVC11 (Visual C++ 2012)

### TS or NTS
phpinfo中
搜索：
Zend Extension Build： API220131226,TS,VC11
PHP Extension Build： API20131226,TS,VC11


## memcache
### 客户端
win客户端扩展下载地址：
[http://windows.php.net/downloads/pecl/releases/memcache/3.0.8/](http://windows.php.net/downloads/pecl/releases/memcache/3.0.8/)

P.S.下载各种win平台php扩展dll:[http://windows.php.net/downloads/pecl/releases/](http://windows.php.net/downloads/pecl/releases/)
### 服务端
1.下载服务端程序
D:\memcached\memcached.exe

2.在终端（也即cmd命令界面）下输入以下命令安装windows服务:
```
D:\memcached>memcached.exe -d install
```

3.再输入下面命令启动：
```
D:\memcached>memcached.exe -d start
```

### 管理工具
memadmin:
[http://www.junopen.com/memadmin/](http://www.junopen.com/memadmin/)


## redis

### 客户端
win客户端扩展下载地址：
[http://windows.php.net/downloads/pecl/snaps/redis/2.2.5/](http://windows.php.net/downloads/pecl/snaps/redis/2.2.5/)

### 服务端
github下载安装包
[https://github.com/MSOpenTech/redis/releases](https://github.com/MSOpenTech/redis/releases)

启动redis
输入命令：redis-server.exe redis.conf (bind的ip地址需要修改本地)

### 管理工具
redis studio
redis desktop manager

## Xdebug
xdebug是可以帮助我们调试脚本的利器，平时在页面输出调试信息（var_dump）的时候可以有高亮显示，非常方便和直观。
### 扩展安装
xdebug官网有个向导页面，可以根据用户自己的phpinfo信息来生成对应的扩展文件提供下载，非常方便。
[https://xdebug.org/wizard.php](https://xdebug.org/wizard.php)

### 配置
php.ini的配置文件
```
[XDebug]
;zend_extension = "\xampp\php\ext\php_xdebug.dll"
;xdebug.profiler_append = 0
;xdebug.profiler_enable = 1
;xdebug.profiler_enable_trigger = 0
;xdebug.profiler_output_dir = "\xampp\tmp"
;xdebug.profiler_output_name = "cachegrind.out.%t-%s"
;xdebug.remote_enable = 0
;xdebug.remote_handler = "dbgp"
;xdebug.remote_host = "127.0.0.1"
;xdebug.trace_output_dir = "\xampp\tmp"
zend_extension = php_xdebug-2.4.0-5.6-vc11.dll
xdebug.var_display_max_children=128
xdebug.var_display_max_data=1024
xdebug.var_display_max_depth=15
```
下面几个配置可以设置var_dump打印的数组层级。

## 其他扩展
打开扩展php_openssl
打开扩展php_curl

# 修改rewrite规则（Apache）
## 开启mod_rewrite模块
在conf目录下httpd.conf中找到
LoadModule rewrite_module modules/mod_rewrite.so
这句，去掉前边的注释符号“#”，或添加这句。

## 允许在任何目录中使用“.htaccess”文件
将“AllowOverride”改成“All”（默认为“None”）
```
# AllowOverride controls what directives may be placed in .htaccess files.
# It can be “All”, “None”, or any combination of the keywords:
# Options FileInfo AuthConfig Limit
#
AllowOverride All
```
## 或在vhosts.conf中直接配置
```
<VirtualHost *:80>
    DocumentRoot "E:/MiaCode/api/"
    ServerName dev.api.com
    ServerAlias api.miyabaobei.com #配置别名
    <IfModule mod_rewrite.c>
   		RewriteEngine on
   		RewriteCond %{REQUEST_FILENAME} !-d
   		RewriteCond %{REQUEST_FILENAME} !-f
   		RewriteRule ^(.*)$ /index.php/$1 [L]
    </IfModule>
</VirtualHost>

```

# 参考资料
xampp: 
http://www.cnblogs.com/BTMaster/p/3705640.html
memcache：
http://blog.csdn.net/wusuopubupt/article/details/9128431
redis:
http://blog.csdn.net/lilien1010/article/details/8710635
http://www.mamicode.com/info-detail-1003539.html
http://blog.csdn.net/musicrabbit/article/details/9953363
rewrite:
http://www.cnblogs.com/njcdh/articles/1772011.html