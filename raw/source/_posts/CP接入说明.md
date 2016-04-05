---
title: CP接入说明
date: 2016-01-13 15:40:36
tags: DHB
---
# CP账号授权&订单推送说明文档

## 概述
1. 适用场景：黄页中接入的第三方服务（H5）页面方式
2. 需要进行账号打通，即服务方需要获取电话邦的账号唯一标识进行业务逻辑的处理，以及支付，生成订单并推送给电话邦。



## 整体开发流程
1. PM进行CP对接前期需求沟通（建立讨论组，明确技术负责人）
2. 对新接入的服务进行名称定义，分配svckey/svcsecret, client_id/clientsecret,应用入口地址，回调地址，logo图标
3. PM开通app内测试入口
3. 在o2o前台增加服务目录，并根据提供的参数和回调地址，完善服务的入口页，并提供给服务方技术人员进行调试
4. 账号对接时进行技术支持
5. 沟通讨论订单推送的相关字段
6. 根据确定的字段，提供订单推送文档
7. 进行订单推送的技术支持（主要是接口的签名调试，订单内容确认等）
8. 开发新的订单模板页
9. 整个服务的集成测试，调试

## 对接需要做的工作
### 分配相关资源和配置
#### svckey/svcsecret分配
分配地址：http://mgmt.dianhua.cn/svckey.php
点击添加key按钮
弹出的页面需要填写payment_type：0，有效期：2016-12-31 00:00:00，是否可用：勾选，对应目录：填写之前和PM确认过的英文服务名称
P.S. 该英文服务名称将用于此服务的目录创建等所有需要此名称的地方，均需保持一致。

#### client_id/clientsecret分配
分配地址：https://apis.dianhua.cn/authorization/addclient.php
需要填写：
1. 应用名称：该应用的中文名称，如：买车险
2. 应用图标：为服务方提供的图标，经过小寒处理过会提交到svn，下载下来，重命名为icon_服务名.png 放在图片服务器的如下目录即可：http://s.dianhua.cn/images/authorization_logo/icon_recharge.png
3. 回调地址：服务方提供的回调地址
4. 服务密钥：即改服务的svckey值

提交后会在数据库中生成一条配置记录，可查看确认（`SELECT * from dict_user_oauth_clients ORDER BY client_id desc`）

### 开发服务相关文件
#### 在o2o前台增加服务目录
需要在线上o2o目录下建立该服务目录，并建立3个文件（config.php，index.php，order_list.php）
**内容如下：**

config.php
```PHP
$svckey='SVCkosf00zPQlKEmErYVASYetUKeyyq8';
$svcname='car_insurance';
$svcsecret='0meRj3BzMa7erMi8u2rapjVzsxfKJlFjOKbXoexafWfwHowchndwxzngwsQf36EsqhqVlDwq91erhbuoEPBcoPiAizhBWNvn621db27aa775848239878237c13252fe';
```
需要更改为对应的服务名称，svckey信息

index.php
```PHP
header('Cache-Control: max-age=86400');
require_once(__DIR__."/config.php");
require_once(__DIR__."/../model/yulorepage.php");
$yp=new yulorepage($svckey,$svcname,$svcsecret);
//Begin Track
require_once(__DIR__.'/../analytics/track_h5.php');
track_h5($svckey);
//End Track
$url='https://mobilesdk.pingan.com.cn/ebusiness/upingan/welcome.html?WT.mc_id=sc03-app-dhb-00001&WT.port_id=01';
header("HTTP/1.1 301 Moved Permanently");
header("Location: $url");
```
其中只需要更改$url的值为服务方提供的入口地址。

order_list.php
```PHP
function car_insurance_order_list_display($order) {
    $statusArr = array(
        '1' => '待处理',
        '2' => '待付款',
        '3' => '已付款',
        '4' => '已取消',
    );
    $rs = <<<EOF
	<div class="orderHistory_main more">
		<p class="orderHistory_content">
			<span class="orderHistory_content_pic"><img alt="" src="//s.dianhua.cn/wap/images/order_details_v1.1/icon_carinsurance.png"></span>
			<span class="orderHistory_content_tit">{$order['desc']}</span>
		</p>
		<p class="orderHistory_content_result">
        	<span class="orderHistory_content_main1">应付金额：{$order["price"]}元</span>
            <span class="orderHistory_content_main2">订单状态：<span class="sub wrapper_col_ff6000">{$statusArr[$order['data']['order_status']]}</span></span>
        </p>
		<p class="orderHistory_content_main">
			<span class="orderHistory_content_main1">订单号：{$order["snid"]}</span>
			<span class="orderHistory_content_main2">{$order["create_time"]}</span>
		</p>
	</div>
EOF;
    if (isset($order['data']['order_url']) && !empty($order['data']['order_url'])) {
        $rs .= '<input type="hidden" value="' . urldecode($order['data']['order_url']) . '" />';
    }
    return $rs;
}
```
该文件的function命名规则为： 服务名+_order_list_display($order)
$order为框架中传递过来的订单内容变量，使用扩展信息的内容可以类似这样：`$order['data']['order_url']`

#### 生成订单推送文档
可参考：`电话帮第三方订单接口文档-平安车险.docx`

#### 订单联调
查看数据表（SELECT * from dict_transaction）数据是否正确，订单在手机上是否显示正常。

## 系统实现说明
### 账号授权流程图
参考：`电话邦账号接入授权流程.png`

### 涉及的数据表
账号授权：
dict_user_oauth_clients 服务方client信息表，存储客户端的id/secret，回调地址，logo等
dict_user_oauth_auth_codes 存储OAuth授权时颁发的code信息
dict_user_oauth_tokens 存储OAuth授权时颁发的token信息，其中uid字段格式为 apikey+下划线+auth_id

订单：
dict_transaction 订单信息表
dict_transaction_history 订单历史记录表

### 涉及的项目代码
账号授权：
`src_portal_api\apis\authorization\` 其中代码一般不需要调整，已支持多家CP

订单推送接口：
`src_portal_api\apis\order\order_create.php` 具体疑问可咨询@宇东

### 涉及的文档目录
文档均在svn中，具体目录：`doc_yp\3.Backend\API\电话帮超级黄页服务方接入（H5）文档`


注意，每次生成好svckey/client_id后，都需要及时在svn中增加新文档，可以参考旧文档模板进行添加 并提交。
svckey:`doc_yp\3.Backend\API\SVCKEYS`
oauth client:`doc_yp\3.Backend\API\OAuthClientIDs`

## 其他注意事项
1. 如果某服务商有多个服务器需要分别进行账号打通，则以上分配的信息需要并列分配多套。（如 平安车险）
2. 签名调试时，如需验证参数，可在接口url中添加get参数&debug=true来打印签名的原始串和加密串，方便对比
3. 后台需要进行log记录的，可以调用公共log组件，生成log文件跟踪
	==调用方式如下==
	```
    define('LOG_ROOT_PATH', __DIR__ . '/'); //引用前定义log文件路径，可以覆盖默认路径
    require_once('common/log/log.class.php'); 
    $logger = \YoloreLog\Logger::getInstance('testLog'); //参数为log的文件名
    $logger->info('log info'); //还有error，debug方法。
    ```
4. 账号打通时，服务方只能在电话邦app中配置的特殊号码入口进行，无法在网页中进行，原因是其中有一部分逻辑是厂商账号登陆验证，需要JSBridge弹出短信验证界面，PC浏览器上无法实现此功能。所以就无法正常写入cookie（authid）













