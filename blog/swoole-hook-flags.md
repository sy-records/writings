# Swoole 一键协程化设置 flags 的问题

从 Swoole4 版本开始，提供了一键协程化的功能，采用 Hook 原生 PHP 函数的方式实现协程客户端，通过一行代码就可以让原来的同步 IO 的代码变成可以协程调度的异步 IO，即一键协程化。

目前有两种方式设置要 Hook 的函数范围：

```php
Swoole\Coroutine::set(['hook_flags'=> SWOOLE_HOOK_ALL]);

Swoole\Runtime::enableCoroutine(SWOOLE_HOOK_ALL);
```

同时从`v4.6.0`版本开始，协程容器中默认开启所有 Hook，即 `SWOOLE_HOOK_ALL`：

```php
use Swoole\Runtime;
use function Swoole\Coroutine\run;

run(function () {
    assert(Runtime::getHookFlags() === SWOOLE_HOOK_ALL);
});
```

那么有两种方式可以设置，具体使用哪种方式来设置呢？

先来看一段代码：

> 基于 Swoole `v4.7.1` 版本，并开启了`--enable-swoole-curl`

```php
use Swoole\Coroutine;
use Swoole\Runtime;

function hook_dump()
{
    var_dump([
        'file' => (Runtime::getHookFlags() & SWOOLE_HOOK_FILE) === SWOOLE_HOOK_FILE,
        'native' => (Runtime::getHookFlags() & SWOOLE_HOOK_NATIVE_CURL) === SWOOLE_HOOK_NATIVE_CURL,
        'curl' => (Runtime::getHookFlags() & SWOOLE_HOOK_CURL) === SWOOLE_HOOK_NATIVE_CURL,
    ]);
}

Coroutine\run(function () {
    hook_dump();

    $flags = SWOOLE_HOOK_ALL ^ (SWOOLE_HOOK_FILE | SWOOLE_HOOK_NATIVE_CURL | SWOOLE_HOOK_CURL);

    Coroutine::set(['hook_flags' => $flags]);
    hook_dump();

    Runtime::enableCoroutine($flags);
    hook_dump();
});
```

是不是觉得应该输出为：

```
array(3) {
  ["file"]=>
  bool(true)
  ["native"]=>
  bool(true)
  ["curl"]=>
  bool(false)
}
array(3) {
  ["file"]=>
  bool(false)
  ["native"]=>
  bool(false)
  ["curl"]=>
  bool(false)
}
array(3) {
  ["file"]=>
  bool(false)
  ["native"]=>
  bool(false)
  ["curl"]=>
  bool(false)
}
```

但实际上输出结果却为：

```
array(3) {
  ["file"]=>
  bool(true)
  ["native"]=>
  bool(true)
  ["curl"]=>
  bool(false)
}
array(3) {
  ["file"]=>
  bool(true)
  ["native"]=>
  bool(true)
  ["curl"]=>
  bool(false)
}
array(3) {
  ["file"]=>
  bool(false)
  ["native"]=>
  bool(false)
  ["curl"]=>
  bool(false)
}
```

这个时候就会以为是`Coroutine::set(['hook_flags' => $flags]);`没生效，是 Bug 吗？ 当然不是 Bug 了。

分开来测试一下：

```php
use Swoole\Coroutine;
use Swoole\Runtime;

function hook_dump()
{
    var_dump([
        'file' => (Runtime::getHookFlags() & SWOOLE_HOOK_FILE) === SWOOLE_HOOK_FILE,
        'native' => (Runtime::getHookFlags() & SWOOLE_HOOK_NATIVE_CURL) === SWOOLE_HOOK_NATIVE_CURL,
        'curl' => (Runtime::getHookFlags() & SWOOLE_HOOK_CURL) === SWOOLE_HOOK_NATIVE_CURL,
    ]);
}

$flags = SWOOLE_HOOK_ALL ^ (SWOOLE_HOOK_FILE | SWOOLE_HOOK_NATIVE_CURL | SWOOLE_HOOK_CURL);
Coroutine::set(['hook_flags' => $flags]);
Coroutine\run(function () {
    hook_dump();

    Coroutine::set(['hook_flags' => SWOOLE_HOOK_ALL]);
    hook_dump();
});
```

再增加一个协程嵌套进行测试

```php
Coroutine\run(function () {
    hook_dump();

    Coroutine::set(['hook_flags' => SWOOLE_HOOK_ALL]);
    go(function () {
        hook_dump();
    });
});
```

输出发现全是`false`，也就是说在协程中通过`Coroutine::set`动态设置 flags 不会生效

```
array(3) {
  ["file"]=>
  bool(false)
  ["native"]=>
  bool(false)
  ["curl"]=>
  bool(false)
}
array(3) {
  ["file"]=>
  bool(false)
  ["native"]=>
  bool(false)
  ["curl"]=>
  bool(false)
}
```

将协程容器中的`Coroutine::set`换为`Runtime::enableCoroutine`进行测试

```php
Coroutine\run(function () {
    hook_dump();

    Runtime::enableCoroutine(SWOOLE_HOOK_ALL);

    hook_dump();

    go(function () {
        hook_dump();
    });
});
```

发现结果是生效的。

```
array(3) {
  ["file"]=>
  bool(false)
  ["native"]=>
  bool(false)
  ["curl"]=>
  bool(false)
}
array(3) {
  ["file"]=>
  bool(true)
  ["native"]=>
  bool(true)
  ["curl"]=>
  bool(false)
}
array(3) {
  ["file"]=>
  bool(true)
  ["native"]=>
  bool(true)
  ["curl"]=>
  bool(false)
}
```

综上所述，可以得到以下结论：

1. `Runtime::enableCoroutine()` 可以在服务启动后 (运行时) 动态设置 flags，调用方法后当前进程内全局生效
2. `Coroutine::set()` 可以理解为 PHP 的 `ini_set()`，需要在 `Server->start()` 前或 `Coroutine\run()` 前调用，否则设置的 `hook_flags` 不会生效
3. 无论是 `Coroutine::set()` 还是 `Runtime::enableCoroutine()` 的方式，都应该只调用一次，重复调用会被覆盖

即 `Runtime::enableCoroutine()`支持动态设置 flags，而`Coroutine::set()`的方式不支持。
