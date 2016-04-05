---
title: DHB-Yulorepages-PUSH平台设计
date: 2016-01-13 15:43:10
tags: DHB
---
# PUSH模块设计 Ver1.0
## 推送流程
**推送流程图**
![push](https://s.dianhua.cn/wap_test/images/doc/push.png)

## 推送平台总体设计
1. 将推送平台建立在现有新版黄页接口中（apis.dianhua.cn/yulorepages/?s=push）
2. 推送相关的接口采用参数字母表签名方式鉴权
3. 各个厂商/平台需要使用推送功能均需调用推送平台的接口
4. 推送规则，推送信息配置目前均为手工配置在数据表中
3. 推送相关的表建立在yulorepages数据库中，以push开头

推送任务分立即推送和延迟推送和批量推送3种。立即推送针对于订单变化的情况，当订单状态变化需要推送时立刻调用推送接口推送消息。延迟推送针对于批量查询类业务，如违章查询或摇号查询。查询任务本身单独实现（可配置在闲时查询），任务将需要推送的信息和推送时间发送到task表中，由后台任务轮询task表，调推送接口推送消息。批量推送也需要定时任务在指定时间触发。




## 预计开发的工作

1. 个推接口调试/sdk集成(开发账号注册，证书配置，测试机device绑定)
2. 服务器/客户端 推送消息模板确定（标题，内容，响铃，跳转链接，详情页）
3. 推送平台框架搭建/数据库设计，创建
3. 服务端接口开发
4. 客户端/服务端/个推 联调
5. 订单接口等其他需要调用推送消息的模块 增加推送功能
6. 后台推送任务（适用于非订单状态变更的推送情景，如给未使用买彩票的用户推送信息等）
7. app服务推送开关管理页开发（web/native）
8. 推送触发模块设计
9. 推送内容配置管理（管理系统）
10. 推送队列&后台任务的设计与实现

## 待确认问题
1. 同一用户登录多个设备，如何进行推送？（多个设备都推？）
2. 用户未登录/激活任何服务，如何上传推送的clientid？ （考虑在服务推送状态接口中进行上传）
3. 服务开关状态以用户为准还是以设备为准？
4. 每个服务是否有默认的推送开关配置？在哪管理？

## 推送平台接口
接口位置：https://apis.dianhua.cn/push/
测试接口：https://apis.dianhua.cn/push_test/
接口返回值：JSON `{"status":"0","msg":"success","data":"info"}`
## 预计增加的接口

### 1.用户设备绑定接口
**方案1：集成在用户激活接口中**

增加参数

| 参数名|是否必填|类型| 说明|
| -------- | --------|--|---|
| device_id | True |string| 手机唯一标识（device_token）|
| push_client_type |True|string|推送渠道（如getui）|
| push_client_id |True|string|推送服务方唯一id|
| client_ext_data |False|string|推送服务方附加信息|
| app_name | True |string| 应用名称，如：电话邦生活黄页|
| mobile_system | True |string| 手机系统类型（IOS，Android）|
| push_status | False |int| 接收消息状态（0接收，1不接收。默认0）|

生活黄页方便集成。

**方案2：新增接口bindUserDevice**

**bindUserDevice**

输入参数

| 参数名|是否必填|类型| 说明|
| -------- | --------|--|---|
| apikey| True |string| 厂商标识 |
| auth_id | True |string| 用户标识|
| device_id | True |string| 手机唯一标识（device_token）|
| push_client_type |True|string|推送渠道|
| push_client_id |True|string|推送服务方唯一id|
| app_name | True |string| 应用名称，如：yolorepage 电话邦生活黄页|
| mobile_system | True |string| 手机系统类型（IOS，Android）|

返回参数

|参数|说明|
|-|-|
|status|绑定的状态（0成功，非0失败）|


### 2.消息推送接口
**push_message**

说明：user_id在注册时可以得知推送渠道（即通过apikey区分）
输入参数

| 参数名|是否必填|类型| 说明|
| -------- | --------|--|---|
| user_ids| True |string| 用户id列表，多个id以下划线分割（最多支持20个用户）|
| content | True |string| 需要推送的数据，JSON格式。具体字段需和客户端商定|


返回参数

|参数|说明|
|-|-|
|status|推送的状态（0成功，非0失败）|

### 3.服务推送状态接口(无需用户登录调用)
**push_svc_config**

说明：上传该设备（用户）使用的每个服务的推送开关配置，如用户未登录，则临时绑定到用户id为0的用户上。
输入参数

| 参数名|是否必填|类型| 说明|
| -------- | --------|--|---|
| device_id | True |string| 手机唯一标识（device_token）|
| push_status | True |string| 服务推送开关 取值为：on全部服务接收，off为全部不接收，如需每个服务单独控制，则取值为JSON字符串（格式：服务id 开关状态  示例=={"123":"on","456":"off"}==）|


返回参数

|参数|说明|
|-|-|
|status|操作的状态（0成功，非0失败）|

### 4.服务列表接口(登录/未登录)
**push_svc_list**

说明：
1. 服务的列表由之前的接口获取（包括服务id，服务名称）
2. 返回每个用户的推送开关配置，如用户未登录，则返回服务默认推送开关配置。
用户注册多个设备如何处理？

输入参数

| 参数名|是否必填|类型| 说明|
| -------- | --------|--|---|
| device_id | False |string| 手机唯一标识（device_token）|
| apikey| True |string| 厂商标识 |
| auth_id | True |string| 用户标识|


返回参数

|参数|说明|
|-|-|
|svc_list|服务列表及默认开关状态，格式：==[{"svc_id":"123","svc_name":"话费充值","push_status":"on"},{"svc_id":"456","svc_name":"违章查询","push_status":"off"}]==|
|user_svc_config|用户服务开关状态on全部服务接收，off为全部不接收，如需每个服务单独控制，则值为JSON字符串（格式：服务id 开关状态  示例`{"123":"on","456":"off"}`）|




