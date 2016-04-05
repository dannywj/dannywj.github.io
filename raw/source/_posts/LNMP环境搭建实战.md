---
title: LNMP环境搭建实战
tags: develop
date: 2016-03-11 10:48:46
---

之前曾经使用过LNMP一键安装包，但是安装后的配置修改起来不是很方便。为了进一步熟悉LNMP架构的配置，今天重新一个组建一个组建的进行安装。
P.S. 安装方式以`apt-get`方式进行


# ubuntu环境准备
首先安装好ubuntu环境，必要的软件，vim等工具安装完毕，接下来更新一下资源列表，保证我们后续的自动获取的是最新的。
`sudo apt-get -y update` 更新源列表，保证源列表是最新的。

# 安装Nginx
```
sudo apt-get install nginx -y
```
安装完毕后，Nginx的配置文件在/etc/nginx目录下。使用以下命令启动Nginx：
```
sudo service nginx start
```

以后修改过nginx的配置以后，需要验证是否有问题，可以使用命令：
```
sudo nginx -t 
```
来验证。正确会返回：
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

```
`我们可以得到nginx的配置文件的路径`

# 安装MySQL
```
sudo apt-get install mysql-server -y
```

我们可以使用以下命令登录MySQL：
```
mysql -uroot -p
```
按提示输入root密码，就会进入MySQL的交互界面，说明已经安装成功。

# 安装PHP
```
sudo apt-get install -y php5 php5-fpm php5-mysql
```
其中包含了php-fpm


# 调试

## 修改nginx配置文件
`
修改nginx默认配置文件：/etc/nginx/sites-available$ sudo vim default
`
主要配置server模块
```
server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        root /usr/share/nginx/html;
        index index.html index.htm;

        # Make site accessible from http://localhost/
        server_name localhost;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                try_files $uri $uri/ =404;
                # Uncomment to enable naxsi on this location
                # include /etc/nginx/naxsi.rules
        }

        location ~ \.php$ {
                fastcgi_pass unix:/var/run/php5-fpm.sock;  #此处需先检查php5-fpm的使用方式，否则报错502！！！！
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME /var/www$fastcgi_script_name;
                include /etc/nginx/fastcgi_params;
        }
}
```
其中的`fastcgi_pass`容易出错，这里主要就是看你的系统的php-fpm使用的是sock，还是9000端口。然后再nginx的配置里面吧解析php的方式改一下就好了。

参考：
```
之前使用ubuntu server 12.04.成功安装了LNMP，并且用得不错。
然后我就在我得本机环境上安装了ubuntu 12.10 desktop版本。。
按照我之前的一篇文章来安装LNMP。
可是等我安装成功之后发现
http://localhost能够正常出现 welcome to nginx 的画面，
然后我就写了个php的探针文件，可是这下报错了。502.。。
这下我就郁闷了。。。为什么呢？
首先来让我们看看之前得配置。关于PHP一块的配置。
location ~ \.php$ { 
    try_files $uri =404; 
    fastcgi_pass 127.0.0.1:9000; 
    fastcgi_index index.php; 
    include fastcgi_params; 
} 
问题其实出现在 fastcgi_pass得配置上面。
在ubuntu 12.10安装了php5-fpm之后。我们可以去
/etc/php5/fpm/pool.d/www.conf 
里面找到这样一条代码：
listen = /var/run/php5-fpm.sock 
这个是使用unix得方式来使用php5-fpm得。
这个时候。我们得php配置要改成
location ~ \.php$ { 
    fastcgi_pass unix:/var/run/php5-fpm.sock; 
    fastcgi_index index.php; 
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name; 
    include fastcgi_params; 
} 
这里fastcgi_pass的地址要改成这个。。这样你的lnmp就正常了。
总结：
这里主要就是看你的系统的php-fpm使用的是sock，还是9000端口。然后再nginx的配置里面吧解析php的方式改一下就好了。
```

root字段的配置是代码存放的路径。

## 重启相关服务
```
sudo service php5-fpm restart 
sudo service nginx restart
```

## 调试代码
mysql test:
```php
<?php
$servername = "localhost";
// 这里填写数据库的用户名和密码
$username = "root";
$password = "juewang";

$conn = new mysqli($servername, $username, $password);

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}
echo "Connected successfully";

$sql='select * from yule.userinfo';
$re=$conn->query($sql);
var_dump($re);

```


# memcache组建安装

## 安装Memcache服务端
```
sudo apt-get install memcached
```
安装完Memcache服务端以后，我们需要启动该服务：
```
memcached -d -m 128 -p 11111 -u root
```

## 安装Memcache客户端（PHP5为示例）
```
sudo apt-get install php5-memcache
```
安装完以后我们需要在php.ini里进行简单的配置,在末尾添加如下内容：
```
[Memcache]
; 是否在遇到错误时透明地向其他服务器进行故障转移。
memcache.allow_failover = On
; 接受和发送数据时最多尝试多少个服务器，只在打开memcache.allow_failover时有效。memcache.max_failover_attempts = 20
;数据将按照此值设定的块大小进行转移。此值越小所需的额外网络传输越多。
; 如果发现无法解释的速度降低，可以尝试将此值增加到32768。
memcache.chunk_size = 8192
; 连接到memcached服务器时使用的默认TCP端口。
memcache.default_port = 11111
; 控制将key映射到server的策略。默认值”standard”表示使用先前版本的老hash策略。
; 设为”consistent”可以允许在连接池中添加/删除服务器时不必重新计算key与server之间的映射关系。
;memcache.hash_strategy = “standard”; 控制将key映射到server的散列函数。默认值”crc32″使用CRC32算法，而”fnv”则表示使用FNV-1a算法。
; FNV-1a比CRC32速度稍低，但是散列效果更好。
;memcache.hash_function = “crc32″
```
重启nginx

## 测试
```php
<?php
error_reporting(E_ALL);
header("Content-type: text/html; charset=utf-8");

$mem = new Memcache; //创建Memcache对象
$mem->connect("127.0.0.1", 11111); //连接Memcache服务器

$val = '这是一个Memcache的测试.';
$key = md5($val);

$mem->set($key, $val, 0, 120); //增加插入一条缓存，缓存时间为120s
if(($k = $mem->get($key))){ //判断是否获取到指定的key
	echo 'from cache:'.$k;
} else {
	echo 'normal'; //这里我们在实际使用中就需要替换成查询数据库并创建缓存.
}

```

## MC客户端
Memcached有方便管理的客户端
`http://www.junopen.com/memadmin/`

另外，平时自己也开发了一个简易的界面：
```php
<?php
/**
 * Created by PhpStorm.
 * User: YULORE-USER
 * Date: 14-8-25
 * Time: 下午12:34
 */

$svckey='SVCTrafficVioxFAegggkx0axss9OtmA';
$svcname='regulation';
$svcsecret='qGAaF5Wffhf1aigfWbHGcjnpXs7AviomQQm5wjB1qGuHqbNn24UWEilB0knNt1zjsCFfnr73zTmpuz2NVxJA6unxeRkxtk7c4f93ceb07699bb2508f9d4c37da1059a';

require_once(__DIR__ . "/../model/yulorepage.php");
$yp = new yulorepage($svckey, $svcname, $svcsecret);
$result = '';
if (isset($_POST['clear_key']) && $_POST['clear_key'] != null) {
    $yp->memcache->delete($_POST['clear_key']);
    echo("<script>alert('clear success')</script>");
}
if (isset($_POST['get_key']) && $_POST['get_key'] != null) {
    $result = $yp->memcache->get($_POST['get_key']);
    if (empty($result)) {
        $result = '<span style="color:red">result is null</span>';
    }
}
if (isset($_POST['clear_all_flag']) && $_POST['clear_all_flag'] != null && $_POST['clear_all_flag'] == 'true') {
    $yp->memcache->flush();
    echo("<script>alert('clear all success')</script>");
}
?>
<html>
<head>
    <title>Memcached clear tool</title>
</head>
<body>
<h2>Memcached clear tool</h2>
<form id="memform" action="index.php" method="POST">
    <p>Clear mem by key</p>
    <p>
        key:<input name="clear_key" type="text"/> <input type="submit" value="clear">
    </p>
    <p>
        <span id="clearresult" style="color:red"></span>
    </p>
    <br>
    <p>Clear all memcached</p>
    <p><input type="button" name="clear_all" value="clear all" onclick="clearAll();">
        <input type="hidden" name="clear_all_flag" id="clear_all_flag">
    </p>
    <br>
    <p>Get mem by key</p>
    <p>key:<input type="text" name="get_key"/> <input type="submit" value="show detail"></p>
    <br>
    <div id="info" style="font-size: 11px;">
        <?php echo $result; ?>
    </div>
</form>
<script>
    function clearAll() {
        if (!confirm('are you sure?')) {
            return false;
        }
        document.getElementById('clear_all_flag').value = 'true';
        document.getElementById('memform').submit();
    }
</script>
</body>
</html>
```

# 总结
各个模块需要通过配置文件关联起来，才能使nginx和php协同工作，需要多尝试。后续会补充redis的配置进来。

# 参考资料

```
https://mos.meituan.com/library/24/how-to-install-lnmp-on-ubuntu/
http://www.cnblogs.com/phppear/archive/2012/11/08/2760030.html
http://leexiaobo.blog.51cto.com/2851883/1074685

http://blog.csdn.net/scelong/article/details/7245343
```