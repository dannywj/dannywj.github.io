---
title: PHP对象方法链式调用
tags: [develop,architecture&design]
date: 2016-08-04 18:27:19
---
# 发现美与丑
今天白天发生了一个DB SQL写法不正规的情况（查询时int字段值加了引号），引起了索引失效，查询速度异常的慢。由此，回顾了一下CI里SQL拼接所用的方式正是链式调用，那么自己所写的类何不也尝试一下这种优雅的方式呢？

# 链式调用的好处
1. 代码书写更加优雅，大大缩减代码的行数。
2. 清晰，方便自由组合

# 示例代码
## 类
```php
<?php

/**
 * 场景主题类(支持链式调用)
 * Created by DannyWang
 * wangjue@mia.com
 * 2016/7/14
 */
class Theme {
    private $_id;
    private $_title;
    private $_subtitle;
    private $_icon;
    private $_type;

    public function __construct() {
    }

    public function getId() {
        return $this->_id;
    }

    public function setId($id) {
        $this->_id = $id;
        return $this;
    }

    public function getTitle() {
        return $this->_title;
    }

    public function setTitle($title) {
        $this->_title = $title;
        return $this;
    }

    public function getSubtitle() {
        return $this->_subtitle;
    }

    public function setSubtitle($subtitle) {
        $this->_subtitle = $subtitle;
        return $this;
    }

    public function getIcon() {
        return $this->_icon;
    }

    public function setIcon($icon) {
        if (!empty($icon)) {
            $this->_icon = $icon;
        }
        return $this;
    }

    public function getType() {
        return $this->_type;
    }

    public function setType($type) {
        $this->_type = $type;
        return $this;
    }

    public function toArray() {
        return array(
            'id' => $this->_id,
            'title' => $this->_title,
            'subtitle' => $this->_subtitle,
            'icon' => $this->_icon,
            'type' => $this->_type,
        );
    }

    function __toString() {
        return json_encode(array(
            'id' => $this->_id,
            'title' => $this->_title,
            'subtitle' => $this->_subtitle,
            'icon' => $this->_icon,
            'type' => $this->_type,
        ));
    }
}
```

## 调用
```php
$theme = new Theme();
// 链式调用设置多个属性并返回最终结果
$result = $theme->setId(123)->setIcon('icon test')->setSubtitle('test_subtitle')->setTitle('test')->setType('api')->toArray();
var_dump($result);

// 重写了__toString方法 使之可以直接输出对象
echo $theme;
```

## 结果
```
E:\DannyCode\study\CallLink\client.php:12:
array (size=5)
  'id' => int 123
  'title' => string 'test' (length=4)
  'subtitle' => string 'test_subtitle' (length=13)
  'icon' => string 'icon test' (length=9)
  'type' => string 'api' (length=3)
  
{"id":123,"title":"test","subtitle":"test_subtitle","icon":"icon test","type":"api"}
```

代码怎么都是写，看你想写成哪样。