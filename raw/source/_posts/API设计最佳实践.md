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
配置不同的key,可以对不同场景进行限制.如限制接口调用次数,限制某个用户的调用次数,限制客户端ip单位时间内的访问次数等

## referer来源验证
### 目的:
防止API被未知来源的调用方调用
### 实现原理
referer原是浏览器发送请求时,携带的链接的页面地址.在APP-API交互模型中,沿用了此机制,服务端验证客户端提交的referer是否是白名单内的,进行数据返回.尽管referer可以伪造,但增加一层验证,也提高了一些入侵门槛.

### 示例代码
``` php
function checkReferer($ref_white_list = [])
	{
		if (isset($_SERVER['HTTP_REFERER'])) {
			$ref = $_SERVER['HTTP_REFERER'];
			if (strpos( $ref, 'http://' ) !== 0 && strpos($ref , 'https://') !== 0) {
				$ref = 'http://' . $ref;
			}
			foreach ($ref_white_list as $item) {
				if (preg_match( "/{$item}/i" , $ref)) {
					return true;
				}
			}
			return false;
		} else {
			return false;
		}
	}
```
## 其他验证
### 手机号验证
可以调用第三方的接口对手机号合法性进行验证,如铜盾,数盟等

### 设备号验证
可以调用第三方的接口对设备id合法性进行验证,如铜盾,数盟等

### IP验证
``` php
class Ip
{
    /**
     * 获取客户端IP地址
     * @return string
     */
    public static function client()
    {
        if (getenv('HTTP_X_FORWARDED_FOR')) {
            $client_ip = getenv('HTTP_X_FORWARDED_FOR');
        } elseif (getenv('HTTP_CLIENT_IP')) {
            $client_ip = getenv('HTTP_CLIENT_IP');
        } elseif (getenv('REMOTE_ADDR')) {
            $client_ip = getenv('REMOTE_ADDR');
        } else {
            $client_ip = isset($_SERVER['REMOTE_ADDR']) ? $_SERVER['REMOTE_ADDR'] : '';
        }
        return $client_ip;
    }

    /**
     * 获取服务器端IP地址
     * @return string
     */
    public static function server()
    {
        if (isset($_SERVER)) {
            if (isset($_SERVER['SERVER_ADDR']) && !empty($_SERVER['SERVER_ADDR'])) {
                $server_ip = $_SERVER['SERVER_ADDR'];
            } else {
                $server_ip = isset($_SERVER['LOCAL_ADDR']) ? $_SERVER['LOCAL_ADDR'] : '';
            }
        } else {
            $server_ip = getenv('SERVER_ADDR');
        }
        return $server_ip;
    }
}

```
# 接口加密
## 加密流程
参数签名->des加密->rsa加密des的key

### 示例代码
```php
/**
	 * 获取解密后的数据
	 * @return bool|string
	 */
	private function getDecodeData()
	{
		//获取私钥
		$pkcs12 = file_get_contents($this->privateKey);
		//rsa解密des的key
		$desKey = Rsa::privatekeyDecode($this->encryptKey, $pkcs12);
		if (empty($desKey)) {
			Log\ILogger::getLogger(__CLASS__,__LINE__)->warning(\Util\Param::getProid(), ['rsa解密失败', $_GET, $_POST]);
			return false;
		}
		//des解密数据
		$desData = NewDes::decrypt($this->secureParams, $desKey);
		if (empty($desData)) {
			Log\ILogger::getLogger(__CLASS__,__LINE__)->warning(\Util\Param::getProid(), ['des解密失败', $_GET, $_POST]);
			return false;
		}
        //解析字符串到数组,不进行decode
		$desDataArr = $this->security->proper_parse_str($desData);
		//验证签名
		if (!$this->verifySign($desDataArr)) {
			Log\ILogger::getLogger(__CLASS__,__LINE__)->warning(\Util\Param::getProid(), ['验证签名错误', $desData, $desDataArr]);
			return false;
		}
		return $desData;
	}

	/**
	 * 校验签名
	 * @param $data
	 * @return bool
	 */
	private function verifySign($data)
	{
		$postSign = $data['sign'];
		ksort($data);
		reset($data);
		unset($data['sign']);
		//拼接url，不进行urlencode
		$str = $this->security->my_http_buid($data);
		$makeSign = md5($str . $this->md5Salt);
		if ($postSign != $makeSign) {
			return false;
		}
		return true;
	}
```
