---
title: 会员Plus开发笔记
tags: [develop,work]
date: 2017-09-20 11:08:14
---

# Plus业务简介
经过一个多月的封闭开发，会员Plus项目终于上线了。回顾整个开发过程，虽然艰辛，但还是收获了不少东西。

Plus是在电商项目基础上引入的身份特权，拥有plus身份的用户可以享受自用省钱，分享赚钱等利益，同时推荐新用户可以升级经理或合伙人。

为开发Plus项目，API重新部署了一套服务端接口（PlusAPI）做为原API的一个服务化接口，供API/Wap端进行内网调用。

服务端调用模式采用客户端（Android/IOS）调用原API，API作为代理调用PlusAPI进行业务逻辑处理。IOS

# 底层架构
## 数据拆分

**表结构拆分**

*创建10个日志记录表*
plus_commission_log（总表）
plus_commission_log_0（分表）
plus_commission_log_1
plus_commission_log_2
……

考虑到高层级的用户拥有的分粉丝众多，粉丝购买商品产生的佣金记录也会很多，因此佣金记录表按用户id维度进行了水平拆分。将用户佣金记录表拆分成10个子表，按照哈希的方式进行对应。

具体拆分及获取方式：
```php
class Plus_tool {
    private $_tableCommLogPrefix = 'plus_commission_log';
    private $_commLogTableNum = 10;

    /**
     * 按用户ID获取佣金日志表名称
     * @param $userId
     */
    public function getCommLogTable($userId) {
        if(defined('COMM_LOG_TABLE_NUM')) {
            $this->_commLogTableNum = COMM_LOG_TABLE_NUM;
        }
        if($this->_commLogTableNum == 0) {
            return $this->_tableCommLogPrefix;
        }

        // 取余
        $seq = fmod(intval($userId) ,$this->_commLogTableNum);
        return $this->_tableCommLogPrefix."_{$seq}";
    }
}
```


**高并发数据插入**

表进行拆分后，数据的录入需要考虑高并发的情况，还需要考虑各个子表的id不能重复。
这里采用**redis的incr方式**生成自增的唯一id。
```php
$commLogIdKey = 'plus_comm_log_id';

$tempLogRow['id'] = $redis->incr($this->commLogIdKey);//如果键不存在,则在执行操作之前将其设置为0
$tempLogRow['user_id'] = $logRow['user_id'];
$plusDb->insert($this->plus_tool->getCommLogTable($logRow['user_id']) ,$tempLogRow);
if($plusDb->affected_rows() == -1) {
    return false;
}
```


**子表数据汇总**

数据水平拆分后，为方便查询，还需要汇总到一张总表中。实现思路是异步写入，实现方案可以是定时脚本或者消息队列。
个人更倾向于写入佣金时入MQ，由MQ统一写入汇总表。实际项目中用定时脚本来实现数据同步。


## 数据表同步
考虑到Plus业务是单独的数据库，且会存在与API主库存在关联查询的情况，因此使用定时脚本方式实现了由plus库向mia库同步数据的功能。（同步用户扩展数据及佣金日志数据）

```php
   /**
     * 同步用户扩展表
     */
    public function syncUserExtension() {
        $this->initLog(self::LOG_FILE_NAME_USER_EXTENSION);
        $check_update_time = date('Y-m-d H:i:s', time() - self::USER_EXTENSION_CHECK_TIME_INTERVAL);
        $sql = "select * from {$this->tableUserExtension} where update_time>'{$check_update_time}' ";
        $plusDb = $this->load->database('plus_write', true);
        $query = $plusDb->query($sql);
        while ($row = $query->unbuffered_row('array')) {// 不缓存结果
            $update_id = $row['id'];
            $this->logInfo("--begin sync id:{$update_id}");
            $this->InsertOrUpdateUserExtension($update_id, $row);
        }
        $this->logInfo("end sync table");
    }

    /**
     * 插入或更新用户扩展信息
     * @param $id plus库id
     * @param $data_row plus库信息
     */
    private function InsertOrUpdateUserExtension($id, $data_row) {
        $apiDb = $this->load->database('api_write', true);
        $where = array(
            'id' => $id,
        );
        // check exists
        $select_result = $apiDb->select('*')->from($this->tableUserExtension)
            ->where($where)->get()->result_array();
        //empty insert
        if (empty($select_result)) {
            $apiDb->set($data_row)->insert($this->tableUserExtension);
            $this->logInfo("*id:{$id} insert new success");
        } else {
            //exists update
            $apiDb->set($data_row)->where($where)->update($this->tableUserExtension);
            if ($apiDb->affected_rows() === 0) {
                $this->logInfo('same data.');
            } else {
                $this->logInfo("*id:{$id} update success");
            }
        }
    }
```
*tips：*
*mysql_unbuffered_query的好处：第一是节省内存，第二是它不用等数据获取完全以后操作，直接可以获取一条数据以后就可以操作。*


## 信息处理机制
佣金日志的表数据变更涉及多个业务模块，如数据统计，发券，短信，站内信。
理想的实现方式是使用MQ的fanout模式对每条佣金日志的变更分别进行处理，实现方式简单，但是需要考虑队列数据丢失后的数据修复问题。

而使用脚本增量统计的方式，需要严格依赖运行时间（每次运行不能够重复处理数据，也不能有漏掉的时间间隔）


# 数据库相关
## 复杂业务场景使用事务
提现场景涉及多个表的读写，多个表的更新必须同时处理正确才不影响后续流程。因此，把提现业务的所有sql操作汇总到一个事务中去统一提交。

事务的处理：
```php
public function createCommCash($uid ,$newExtractList,$commList,$open_id){
    //验证用户ID、佣金计税集合、$commList是否为空
    if(empty($uid) || empty($newExtractList) || empty($commList)) {
        return false;
    }
    //加载plus数据库句柄
    $plusWrite = $this->load->database('plus_write' ,true);
    //加载日志插件
    $this->load->library('monolog_client');
    $this->load->library('plus_tool');
    //获取当前时间
    $nowTime = date("Y-m-d H:i:s");

    //开启事物
    $plusWrite->trans_begin();

    /*------------------------插入提现单  开始-------------------------------*/
    //组装插入提现数据
    $cash_code = $this->plus_tool->createCashCode();
    $insertCashData = array(
        'cash_code'=> $cash_code,  //提现单号
        'cash_amount'=>$newExtractList['comm_amount'],           //提现金额
    );

    //执行插入操作
    $plusWrite->insert($this->tableCommCash, $insertCashData);
    $this->monolog_client->info("插入提现申请数据sql:".$plusWrite->last_query(), 'plus_commission_cash');

    //验证是否操作成功
    if ($plusWrite->affected_rows() < 1) {
        //回滚
        $plusWrite->trans_rollback();
        return false;
    }
    /*------------------------插入提现单   结束-------------------------------*/
    /*------------------------business-------------------------------*/
    // 事务提交
    if($plusWrite->trans_status() === FALSE) {
        //回滚
        $plusWrite->trans_rollback();
        return false;
    } else {
        //提交
        $plusWrite->trans_commit();
        $this->monolog_client->info("更新提现数据完成，事务提交success", 'plus_commission_cash');
        //组装返回数据
        $result = array(
            'arrival_price' => floatval($newExtractList['comm_amount']) - floatval($newExtractList['comm_taxation']),  //到账金额
            'taxation_price' => floatval($newExtractList['comm_taxation']),    //税费金额
            'extract_code' => $cash_code,                                         //税费单号
            'extract_id' => $insertId,
            'extract_time' => $insertCashData['apply_time'],                    //提现时间
            'extract_status' => 2,                                               //提现状态
        );
        return $result;
    }
}
```

## SQL性能评估
查询类接口中包含复杂的查询sql，优化时要进行explain慢查询评估，观察索引的使用情况，做合理优化。


## 主从连接
- model的构造方法中不要提前连接数据库句柄，虽然方便，会增加一次写库的连接开销。
- 脚本相关的数据库连接尽量使用主库连接，防止出现数据不同步的问题难以排查。

# 脚本相关
增量统计脚本的运行时间和脚本处理的数据时间段应该配置好。由于增量统计脚本不能重复跑数据，可以考虑每小时的第10分钟运行一次，扫描上一个整小时的待处理数据。
```
每小时第20分钟执行1次
20 * * * * php /crontab/DailyStatistics/syncUserIncome > /dev/null 2>&1
```

增量统计脚本在凌晨可以考虑增加**全量计算**脚本，不影响用户数据。

**脚本及队列需要考虑异常情况的数据修复**

1. 记录错误日志，并用另外的脚本执行修复
1. 队列重新插入，按正常流程处理
1. 记录上次的操作id，后续重复执行



**错误日志处理**
考虑支持从log中检查错误信息，并将错误数据处理并修复，并有重试机制。（如重复2次依然无法修复，进行错误报警）

**脚本防断评估**
脚本需要考虑执行中断的情况，如何后续运行后修补断开时间内的数据。可以使用记录上次完成时间的方式来处理

*处理示例：*
```php
/**
 * 同步用户收益金额
 */
public function syncUserIncome() {
    $this->initLog(self::LOG_FILE_NAME_INCOME);
    // 计算当前时间前整5min的数据
    $begin_time = date('Y-m-d H:i:00', strtotime('-5 minute'));
    $end_time = date('Y-m-d H:i:00');

    // 初始化redis
    $redis = $this->plus_tool->initCronRedis('syncUserIncome');
    if ($redis) {
        $last_end_time = $redis->get($this->syncUserIncomeKey);
        if (!empty($last_end_time)) {
            $begin_time = $last_end_time;
        }
    }

    // business

    $redis->set($this->syncUserIncomeKey ,$end_time);
}
```

如此处理，即使脚本执行频率增大或减小，都不会重复处理数据和遗漏处理数据！


**脚本执行时间配置**
脚本操作同一个表频繁，容易锁表,可适当延迟交错脚本执行时间。
crontab 可以用sleep来进行秒级间隔运行。

```
*/2 * * * * sleep 6;  php /crontab/SyncData/syncCommissionLog/1 > /dev/null 2>&1
*/2 * * * * sleep 12; php /crontab/SyncData/syncCommissionLog/2 > /dev/null 2>&1
*/2 * * * * sleep 18; php /crontab/SyncData/syncCommissionLog/3 > /dev/null 2>&1
*/2 * * * * sleep 24; php /crontab/SyncData/syncCommissionLog/4 > /dev/null 2>&1
*/2 * * * * sleep 30; php /crontab/SyncData/syncCommissionLog/5 > /dev/null 2>&1
*/2 * * * * sleep 36; php /crontab/SyncData/syncCommissionLog/6 > /dev/null 2>&1
*/2 * * * * sleep 42; php /crontab/SyncData/syncCommissionLog/7 > /dev/null 2>&1
*/2 * * * * sleep 48; php /crontab/SyncData/syncCommissionLog/8 > /dev/null 2>&1
*/2 * * * * sleep 54; php /crontab/SyncData/syncCommissionLog/9 > /dev/null 2>&1
```

# 接口相关
## 身份验证相关
- 外网访问的接口，需要严格进行身份验证，接口签名等安全验证机制。如API，wap页等。
- 内网调用的接口（如服务化接口，电视大屏接口，Thrift接口）由于不暴露在外网环境，不需验证用户身份，直接调用即可。


- account/detail用户详细信息接口之前缓存了整个接口数据， 新增的用户类型信息不适合缓存，因此把总缓存改为业务缓存，根据具体的模块信息单独进行缓存。

- 接口配置中，增加业务总开关，整体控制某个业务的开关。

- 详情页针对某个业务的特殊功能，在用户详情信息中增加字段会需要查寻上级信息，代价大，考虑走单独接口，不要使用基础接口。

- 提现接口防重机制
对于重点接口，应该防止短时间内重复调用。可使用redis记录key 成功后删除，10s内操作中防止重复调用。

# 第三方对接
提现调试中签名错误有可能是用的key不对。

**支付异常的处理**
第三方支付失败的情况需要考虑全，涉及如下几种情况：

1. 第三方直接明确返回支付失败
2. 第三方返回成功，实际却支付失败了
3. 第三方返回失败，实际却成功了
4. 调用接口超时，无法接收到成功失败的信息

需要根据不同情况分别处理。一般支付平台都会提供异步通知或支付结果查询的接口，方便进行二次校验。

*处理原则：*

1. 不应该相信同步接口返回的结果，要以异步通知或异步查询结果为准。
2. 支付失败要有数据回退方案。
3. 接口超时可视为操作成功，后续用异步接口进行验证后，如果失败，执行回退方案。

## 支付场景与事务处理
由于支付与写流水表间存在相互依赖关系，但支付又是调用第三方的操作，因此在事务处理中，应该排除进行第三方支付接口的调用逻辑。
将第三方支付的动作放在事务外，如果失败，改写支付结果数据。

**tips：第三方支付时，有可能会出现支付成功后，curl断开的情况。如果放在事务中，会导致回滚而重复付款的情况。**








