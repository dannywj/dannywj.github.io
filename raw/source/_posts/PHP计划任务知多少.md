---
title: PHP计划任务知多少
tags: work
date: 2016-05-04 16:06:21
---
# 代码设计
## 计划任务方案设计

1. 需要有中断后重新运行继续处理的能力。
1. 大数据表数据处理时采用备份表+rename方式快速恢复。
1. 集成在现有框架中，方便DB的配置和其他公共方法的调用。


## 分片处理思维
1. 根据id划分分组，每次读取一组的数据到array中处理。
1. 分组的id可写入到文件中，采用永久循环方式对id文件进行更新和读取。
1. 多台服务器分别执行不同区块内的脚本。要求脚本需要支持输入参数
1. 日志文件记录分大小分割成多个文件存储
1. 没有更新及其他操作时也可以使用`mysql_unbuffered_query`的方式获取数据`mysql_use_result`

## 脚本的参数输入
*几种方式：*
**A.argv变量方式**
```
print_r($argv);
D:\>\wamp\bin\php\php5.3.0\php.exe  \tools\index.php bac ddd
Array
(
    [0] => \tools\index.php
    [1] => bac
    [2] => ddd
)
```
**B.自定义参数方式**
```
// 接收输入参数
$args = getopt('', array('min_id:', 'max_id:'));
if (!isset($args['min_id'], $args['max_id'])) {
        echo 'invalid input params';
        die;
}
$min_id = intval($args['min_id']);
$max_id = intval($args['max_id']);

调用：D:\>\wamp\bin\php\php5.3.0\php.exe  \tools\index.php --min_id "111" --max_id "222"
```
**C.使用CI框架通过脚本调用**
```
/usr/local/php/bin/php /data/www/api/bin/sh.php /ios_activation_task/index/0/100
```

# 服务器方面
1. 关注任务运行时内存的消耗
2. 关注相关php进程是否在执行任务
3. 设置脚本最大执行时间及最大消耗内存
```
set_time_limit(3600 * 24);
ini_set('memory_limit','500M');
```

# 扩展&完善
执行一段时间后，停止，重新运行。防止长时间执行资源消耗过大，效率变慢。

# 示例脚本
A 单独执行的php脚本
```php
<?php
/**
 * ActivationTask
 * Created by DannyWang
 * wangjue@mia.com
 * 2016-04-28
 */

date_default_timezone_set('Etc/GMT-8');
header('Content-Type: text/html; charset=utf-8');
set_time_limit(0);

$push_task = new ActivationTask();
$push_task->run();

class ActivationTask {
    // 全局配置
    const TABLE_IOS_ACTIVATION_LOG = 'activation_logs_copy';
    const FILE_OPERATION_LOG = 'activation_task';
    const FILE_LAST_ID = 'last_id.txt';
    const LOG_SEPARATE_COUNT = 500000;
    const EVERY_PROCESS_COUNT = 2000;

    private $mysqli = '';
    private $counter = 0;

    //初始化，连接DB
    function __construct() {

        $db_config = array(
            'host' => 'localhost',
            'port' => '3306',
            'username' => 'root',
            'passwd' => 'root',
            'dbname' => 'mia',
        );

        $this->mysqli = new mysqli($db_config['host'], $db_config['username'], $db_config['passwd'], $db_config['dbname'], $db_config['port']);
        $this->mysqli->set_charset("utf8");
    }

    //运行主程序
    public function run() {
        // 接收输入参数
        $args = getopt('', array('min_id:', 'max_id:'));
        if (!isset($args['min_id'], $args['max_id'])) {
            echo 'invalid input params';
            die;
        }
        $min_id = intval($args['min_id']);
        $max_id = intval($args['max_id']);

        file_put_contents(__DIR__ . '/' . self::FILE_LAST_ID, $min_id);

        echo '==============begin ActivationTask: [' . date('Y-m-d H:i:s') . ']================';
        $this->log('==============begin ActivationTask: [' . date('Y-m-d H:i:s') . ']================');
        $this->log('==begin process==');

        while (true) {
            $last_id = file_get_contents(__DIR__ . '/' . self::FILE_LAST_ID);
            if (!$last_id) {
                $this->log('last_id is null');
            }

            $sql = "select * from " . self::TABLE_IOS_ACTIVATION_LOG . " where id>{$last_id} and id<{$max_id}
                    ORDER BY id asc limit " . self::EVERY_PROCESS_COUNT;

            $result = $this->select($sql);

            if (empty($result)) {
                $this->log('==end process==');
                $this->log('==============finish ActivationTask================');
                exit("\r\n".'task end.');
            }

            $key = count($result) - 1;
            file_put_contents(__DIR__ . '/' . self::FILE_LAST_ID, $result[$key]['id']);


            foreach ($result as $row) {
                $this->counter++;
                $this->log("current data [num:{$this->counter} id:{$row['id']},idfa:{$row['idfa']}]");

                // 当前扫描的数据行 A
                $idfa = $row['idfa'];
                $openudid = $row['openudid'];
                $device_token = $row['device_token'];
                // 查找重复数据
                $repeat_data = $this->searchRepeatDataByOpenUDID($openudid);
                if (count($repeat_data) == 1) {
                    $repeat_data = $this->searchRepeatDataByIDFA($idfa);
                    if (count($repeat_data) == 1) {
                        $this->log("not find repeat data");
                        continue;
                    }
                }
                // 获取需要保留的数据
                $true_data = array_shift($repeat_data);//B
                $this->log("exists repeat data! saved:", $true_data);

                // 删除多余数据
                foreach ($repeat_data as $val) {//C
                    $delete_id = $val['id'];
                    $this->deleteRepeatData($delete_id);
                }
                $this->log("current process finish");
            }

        }
    }

    private function searchRepeatDataByIDFA($idfa) {
        $sql = "SELECT * from " . self::TABLE_IOS_ACTIVATION_LOG . " where idfa='{$idfa}' ORDER BY user_id desc,created asc ";
        $result = $this->select($sql);
        return $result;
    }

    private function searchRepeatDataByOpenUDID($openudid) {
        $sql = "SELECT * from " . self::TABLE_IOS_ACTIVATION_LOG . " where openudid='{$openudid}' ORDER BY user_id desc,created asc ";
        $result = $this->select($sql);
        return $result;
    }


    private function deleteRepeatData($id) {
        $sql = "delete from " . self::TABLE_IOS_ACTIVATION_LOG . " where id={$id}";
        $result = $this->mysqli->query($sql);
        $this->log('delete:' . $sql);
        return $result;
    }

    //查询方法
    public function select($sql, $key = '') {
        $result = $this->mysqli->query($sql);
        $lists = [];
        while ($row = $result->fetch_array(MYSQLI_ASSOC)) {
            if (!empty($key)) {
                $lists[$row[$key]] = $row;
            } else {
                $lists[] = $row;
            }
        }
        $result->free();
        return $lists;
    }

    //写入日志
    private function log($str, $arr = null) {
        $log_num = floor($this->counter / self::LOG_SEPARATE_COUNT);
        if (is_array($arr)) {
            $arr = json_encode($arr);
        }
        $data = "[" . date('Y-m-d H:i:s') . "]" . $str . ' ' . $arr . "\r\n";
        //echo $str . ' ' . $arr . "\r\n<br/><br/>";
        file_put_contents(__DIR__ . '/' . self::FILE_OPERATION_LOG . '_' . $log_num . '.log', $data, FILE_APPEND);
    }
}
```
B 基于CI框架的php脚本
```php
<?php
/**
 * ActivationTask
 * Created by DannyWang
 * wangjue@mia.com
 * 2016-04-28
 */
if (!defined('BASEPATH')) exit('No direct script access allowed');

class ios_activation_task extends CI_Controller {
    // 全局配置
    const TABLE_IOS_ACTIVATION_LOG = 'ios_activation_logs_copy';
    const FILE_OPERATION_LOG = 'activation_task';
    const FILE_LAST_ID = 'last_id.txt';
    const LOG_SEPARATE_COUNT = 500000;
    const EVERY_PROCESS_COUNT = 1000;
    const BASE_PATH = '/tmp/';
    private $counter = 0;

    //初始化，连接DB
    function __construct() {
        set_time_limit(3600 * 24);
        ini_set('memory_limit','500M');
        parent::__construct();
        $this->db = $this->load->database('log_write', true);
    }

    //运行主程序
    public function index($min_id, $max_id) {

        file_put_contents(self::BASE_PATH . self::FILE_LAST_ID, $min_id);

        echo '==============begin ActivationTask: [' . date('Y-m-d H:i:s') . ']================';
        $this->log('==============begin ActivationTask: [' . date('Y-m-d H:i:s') . ']================');
        $this->log('==begin process==');

        while (true) {
            $last_id = file_get_contents(self::BASE_PATH . self::FILE_LAST_ID);
            if (!$last_id) {
                $this->log('last_id is null');
            }

            $sql = "select * from " . self::TABLE_IOS_ACTIVATION_LOG . " where id>{$last_id} and id<{$max_id}
                    ORDER BY id asc limit " . self::EVERY_PROCESS_COUNT;

            $result = $this->select($sql);

            if (empty($result)) {
                $this->log('==end process==');
                $this->log('==============finish ActivationTask================');
                exit("\r\n" . 'task end.');
            }

            $key = count($result) - 1;
            file_put_contents(self::BASE_PATH . self::FILE_LAST_ID, $result[$key]['id']);


            foreach ($result as $row) {
                $this->counter++;
                $this->log("current data [num:{$this->counter} id:{$row['id']},idfa:{$row['idfa']}]");

                // 当前扫描的数据行 A
                $idfa = $row['idfa'];
                $openudid = $row['openudid'];
                $device_token = $row['device_token'];
                // 查找重复数据
                $repeat_data = $this->searchRepeatDataByOpenUDID($openudid);
                if (count($repeat_data) == 1) {
                    if (empty($idfa)) {
                        continue;
                    }
                    $repeat_data = $this->searchRepeatDataByIDFA($idfa);
                    if (count($repeat_data) == 1) {
                        $this->log("not find repeat data");
                        continue;
                    }
                }
                // 获取需要保留的数据
                $true_data = array_shift($repeat_data);//B
                $this->log("exists repeat data! saved:", $true_data);

                // 删除多余数据
                foreach ($repeat_data as $val) {//C
                    $delete_id = $val['id'];
                    $this->deleteRepeatData($delete_id);
                }
                $this->log("current process finish");
            }

        }
    }

    private function searchRepeatDataByIDFA($idfa) {
        $sql = "SELECT * from " . self::TABLE_IOS_ACTIVATION_LOG . " where idfa='{$idfa}' ORDER BY user_id desc,created asc ";
        $result = $this->select($sql);
        return $result;
    }

    private function searchRepeatDataByOpenUDID($openudid) {
        $sql = "SELECT * from " . self::TABLE_IOS_ACTIVATION_LOG . " where openudid='{$openudid}' ORDER BY user_id desc,created asc ";
        $result = $this->select($sql);
        return $result;
    }


    private function deleteRepeatData($id) {
        $sql = "delete from " . self::TABLE_IOS_ACTIVATION_LOG . " where id={$id}";
        $result = $this->db->query($sql);
        $this->log('delete:' . $sql);
        return $result;
    }

    //查询方法
    public function select($sql, $key = '') {
        $lists = $this->db->query($sql)->result_array();
        return $lists;
    }

    //写入日志
    private function log($str, $arr = null) {
        $log_num = floor($this->counter / self::LOG_SEPARATE_COUNT);
        if (is_array($arr)) {
            $arr = json_encode($arr);
        }
        $data = "[" . date('Y-m-d H:i:s') . "]" . $str . ' ' . $arr . "\r\n";
        //echo $str . ' ' . $arr . "\r\n<br/><br/>";
        file_put_contents(self::BASE_PATH . self::FILE_OPERATION_LOG . '_' . $log_num . '.log', $data, FILE_APPEND);
    }
}
```

P.S. 测试环境修改分支

1. vim exportApi.sh 
1. 修改checkout 分支名称
1. 执行`sh exportApi.sh `