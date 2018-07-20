---
title: API设计最佳实践
tags: develop
date: 2018-07-20 10:10:58
---
在API开发过程中，自己通过参考项目框架的代码，慢慢梳理了几个在API设计过程中需要考虑的问题及实现方式。让接口的安全性，健壮性得到进一步的提升。
# API接口防刷机制
## 回放攻击锁机制
### 目的:
防止API短时间内被连续调用多次
### 实现原理
利用redis的setnx特性(如果key不存在，则 SET,已存在则会设置失败),配合key的毫秒超时时间
### 示例代码
``` php
function getLockMs($key, $timeoutMs = 2000)
{
	// 当key不存在的时候才能设置成功,如果key已存在,设置失败,方法返回true
	if (!$this->_redisClient->setnx($key, 1)) {
		return true;
	} else {
		// key之前不存在,设置成功后,再设置key的超时时间(PEXPIRE为毫秒超时设置)
		$this->_redisClient->PEXPIRE($key, $timeoutMs);
		return false;
	}
}

// 客户端调用
if(getLockMs('api',2000)){
    exit('接口访问太快了!');
}
```
### 使用场景延伸
配置不同的key,可以对不同场景进行限制.如限制接口调用次数,限制某个用户的调用次数等

# 接口加密
