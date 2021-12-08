# 给你的项目增加 Git WebHooks

之前写过 [《使用 Github 的 WebHooks 实现生产环境代码自动更新》](https://qq52o.me/2482.html) ，是将 WebHooks 用于自动部署。<!--more-->

文章中说到了：GitHub、GitLab、Gitee 虽然都是 Git 仓库平台，但是发送的 WebHooks 请求的数据格式有些差别。

那么如何解决这个问题呢？

使用 [sy-records/webhooks](https://github.com/sy-records/webhooks) 的 composer 扩展包，可以让你的项目支持 WebHooks，并且可以自定义 WebHooks 的规则。

例如，你可以指定分支、Tag、提交人、提交内容等条件，来执行一些事件。

同时也可以验证是否为有效的 WebHooks 请求。

## 安装

> 需要 PHP >= 7.2，低版本的建议升级。。。

```bash
composer require sy-records/webhooks
```

## 使用

实例化 `Payload` 对象，获取到对应的 `handler`：

```php
use Luffy\WebHook\Payload;
use Luffy\WebHook\Interfaces\HandlerInterface;

$payload = new Payload();
/** @var HandlerInterface $handler */
$handler = $payload->getHandler();
```

如果存在实现 `MessageInterface` 的 `request` 对象，可以在实例化 `Payload` 时传入，否则的话是从全局变量中获取。

然后就可以操作一些方法了，例如：

```php
// 是否为 ping 请求
$handler->isPing();

// 获取 hook 事件名称
$handler->getHookName();

// 验证是否为有效的 WebHooks 请求
$handler->check($secret);
```

完整的方法可以查看 [HandlerInterface](https://github.com/sy-records/webhooks/blob/master/src/Interfaces/HandlerInterface.php) 接口。
