# MQTT 怎么在单独一个端口上分别使用 v3.x 和 v5.0 协议解析？

MQTT 有 3 个常用的协议等级：v3.1、v3.1.1 和 v5.0，那么如何在一个端口上同时处理 3 种协议等级的解析呢？

例如在 1883 端口上，同时处理 v3.1、v3.1.1 和 v5.0 这 3 种协议等级

simps/mqtt 提供了 MQTT 协议解析的能力，这种需求在之前的版本中也是可以实现的，不过比较麻烦，可能需要这样：

```php
use Simps\MQTT\Protocol\V3;
use Simps\MQTT\Protocol\V5;

$server->on('receive', function (Swoole\Server $server, $fd, $from_id, $data) {
    try {
        $data = V3::unpack($data);
    } catch (\Throwable $e) {
        try {
            $data = V5::unpack($data);
        } catch (\Throwable $e) {
            throw $e;
        }
    }
    var_dump($data);
});
```

解析两次数据来进行尝试获取，代码不够优雅

那么现在呢，很简单。安装 simps/mqtt 最新版 `v1.4.0`，增加了一个`getLevel`的方法

- 使用 composer 加载 simps/mqtt

```bash
composer require simps/mqtt
```

- 创建一个 Server

```php
use Simps\MQTT\Protocol\Types;
use Simps\MQTT\Protocol\V3;
use Simps\MQTT\Protocol\V5;
use Simps\MQTT\Tools\UnPackTool;
use Simps\MQTT\Protocol\ProtocolInterface;

$server = new Swoole\Server('127.0.0.1', 1883, SWOOLE_BASE);

$server->set(
    [
        'open_mqtt_protocol' => true,
        'worker_num' => 1,
        'package_max_length' => 2 * 1024 * 1024,
    ]
);

$server->on('connect', function ($server, $fd) {
    echo "Client #{$fd}: Connect.\n";
});

$server->on('receive', function (Swoole\Server $server, $fd, $from_id, $data) {
    $type = UnPackTool::getType($data);
    if ($type === Types::CONNECT) {
        $level = UnPackTool::getLevel($data);
        $class = $level === ProtocolInterface::MQTT_PROTOCOL_LEVEL_5_0 ? V5::class : V3::class;
        $server->fds[$fd] = ['level' => $level, 'class' => $class];
    }
    /** @var ProtocolInterface $unpack */
    $unpack = $server->fds[$fd]['class'];
    var_dump($unpack::unpack($data));
});

$server->on('close', function ($server, $fd) {
    unset($server->fds[$fd]);
    echo "Client #{$fd}: Close.\n";
});

$server->start();
```

这样代码就看起来简单多了，使用`getType`获取当前包的类型，在`connect`类型的时候获取使用协议类型是什么，

然后存到`$server->fds`中，下文就可以从直接取对应的协议解析类来进行处理。

```php
$type = UnPackTool::getType($data);
if ($type === Types::CONNECT) {
    $level = UnPackTool::getLevel($data);
    $class = $level === ProtocolInterface::MQTT_PROTOCOL_LEVEL_5_0 ? V5::class : V3::class;
}
```

五行代码就可以实现这个功能需求~ 如果你被加鸡腿了不要忘了我哦 :)

分享一个讲解 MQTT 协议的 PPT，你可以不限速下载 🚀

链接：[https://www.aliyundrive.com/s/YUW7P2aDQZU](https://www.aliyundrive.com/s/YUW7P2aDQZU)
