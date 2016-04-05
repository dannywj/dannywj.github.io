---
title: PHP代码规范
date: 2016-01-13 15:43:49
tags: develop
---
# php代码规范 #

2014/8/27 

----------
## 排版 ##

- 缩进使用tab（4个空格）

- if，else即使条件只有一行，也要加上花括号。不允许写成一行。
```
if(true){
	$val=1;
}else{
	$val=2;
}
```
- 初始化array如果采用多行结构时，数据项部分需要缩进，且最后一个数据项后面的逗号不可省略。
*说明：这样做在修改代码增加数据项的时候不容易出现语法错误。*
```
$a = array(
	'a' => 'b',
	'b' => 'c',
	'c' => 'd',
);
```

## 命名 ##
- 普通变量以小写加下划线方式命名
```
$name='danny';
$unique_id=123;
```
- 全局变量以g\_开头。
*说明:全局变量对代码影响很大，以 g\_开头变能在代码中一眼看出是全局变量。*


- 常量命名使用全部大写字符，单词之间以'_'连接。
*如:PAGE_NUM*


- 对于代码中的常量，必须用常量或define表示，不允许直接写在代码中。
```
define('PAGE_NUM', 3);
```

- 私有函数命名需加上'_'前缀。
```
private function _myPrivateFunc() {
}
```

- 类方法命名采用驼峰式命名, 普通方法采用过程函数风格（小写下划线）命名。
```
//类method:
public function getName() {
}
//普通function:
function show_me_the_money() {
}
```
- 文件(除了类)命名使用小写字母，单词之间以'_'连接。
*如:show_lemma.php*

- 配置文件的名称为 配置文件名 + .config.php, 不涉及类的都小写通过”_”连接。
如:dianhua_version.config.php


## 编码 ##
- 对于函数返回值的判断，特别是true/false, 用===/!== 而不是==/!=。

- 除非特殊情况，否则不允许使用require和include,而使用对应的require_once/include_once。

- 每个类单独为一个文件, 文件名为 原类名 + .class.php。
*说明: 利于管理，逻辑清楚，方便autoload等。
例如: 
apis/o2o/buycar/Filter.class.php*


- 函数允许使用默认参数,但是默认参数需要放到参数列表最后面。

- 调用第三方的链接地址，apikey，用户名密码等配置信息不要写死在程序中，要写到配置文件中。

- 用户的相关输入涉及数据库操作、文件操作等敏感操作时需要对输入做专门的转换。
*例如: 
数据库操作中数字型的需要做intval转换，字符串类型的需要通过mysql_real_escape_string过滤。*

- 将php配置中的 register_globals 设置为 Off。
*register_globals允许php将$_GET，$_POST，$_COOKIE等变量里的内容自动注册为全局变量，如果程序里的某些变量没有经过初
始化而直接使用将导致安全问题。*

- 所有sql关键词全部大写，比如SELECT,UPDATE,FROM,ORDER,BY等。



# 前端JS代码规范 #

- 省略ur地址中的 http: 或 https: 的部分
```
<link href="//s.dianhua.cn/wap/css/webapps.css" type="text/css" rel="stylesheet" />
<script type="text/javascript" src="//s.dianhua.cn/wap/js/jquery.js"></script>
```
html页面使用Html5的文件头声明
```
<!DOCTYPE html>
```

- 声明变量必须加上 var 关键字.
- 常量的命名: NAMES_LIKE_THIS, 即使用大写字符, 并用下划线分隔
- 变量，方法，函数采用驼峰式命名
* functionNamesLikeThis, variableNamesLikeThis, ClassNamesLikeThis, EnumNamesLikeThis, methodNamesLikeThis*
- 全局变量使用g前缀
```
var gPageFlag=true;
```
- 语句结束后总是使用分号
- 对于值的判断，特别是true/false, 用===/!== 而不是==/!=。
- eval()方法不要轻易使用
*获取ajaxJSON对象的时候 使用JSON.parse()方法来代替。*

- 在全局作用域上, 使用一个唯一的, 与工程/库相关的名字作为前缀标识.
比如, 你的工程是 "Project Buycar", 那么命名空间前缀可取为 Buycar.*.
```
var Buycar = {};
Buycar.buy = function() {
  //...
};
```
- 即使条件只有一行，也要加上花括号
```
if (something) {
  // ...
} else {
  // ...
}
```

# 其他相关规范 #
- 代码提交到SVN时需要填写注释，说明一下本次修改的内容。
*可以方便他人了解代码修改的历史，已经帮助自己回忆之前所做的修改*
- 提交代码应该删除自己调试时打印变量的相关代码。
- 对于复杂逻辑要有注释标记，说明整段代码或方法实现的功能。
- 对于页面中多次出现的标记变量，如果逻辑复杂，要在声明时注释说明标记变量各种取值的含义。
- 页面代码统一采用utf-8方式编码。