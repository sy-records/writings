# 解决 Swoole 服务报错 Too many open files 文件句柄超出系统限制

如果在 Swoole 的日志中遇到了 `Too many open files` 这种报错，不要慌，在开发 TCP 网络应用的过程中，经常会遇到`Too many open files`这个问题。

这说明你的程序以达到 Linux 所允许的打开文件数上限。需要修改 `ulimit` 设置

可以使用`lsof`查看进程打开的文件句柄数：

```bash
# 将 PID 修改为你要检查的进程ID
lsof -p PID | wc -l
```

也可以去掉`| wc -l`来检查服务打开了那些文件句柄，未进行关闭。

可以使用`ulimit -n`查看当前设置为多少

```shell
[root@shenyan ~]# ulimit -n
100001
```

如果过小，需要进行调整，Swoole 文档中推荐 `ulimit -n` 要调整为 `100000` 甚至更大。

那么如何修改呢？

命令行下执行 `ulimit -n 100000` 即可修改。

但是需要注意，有时候这种修改方法仅当前终端有效，关闭或者重新开个一个终端就会恢复之前的设置。

这时就需要修改 `/etc/security/limits.conf`，加入

```conf
* soft nofile 262140
* hard nofile 262140
root soft nofile 262140
root hard nofile 262140
* soft core unlimited
* hard core unlimited
root soft core unlimited
root hard core unlimited
```

修改 `limits.conf` 文件后，需要重启系统生效。

这样操作完成以后，如果还是报错 `Too many open files`，那么就可以试试检查运行中的进程限制：

```bash
# 将 PID 修改为你要检查的进程ID
cat /proc/PID/limits
```

如果这里的`Max open files`过小，也是需要进行修改的。

这种情况大多数出现于使用`supervisor`等工具进行管理的时候，`supervisor`启动服务默认的`minfds`配置是`1024`，所以会出现 `Too many open files`。

使用`systemd`的话，也是同样的道理，需要修改`LimitNOFILE`。如果不设置或者设置为`LimitNOFILE=unlimited`（不识别），则默认为`1024`。

而这里也要注意设置`LimitNOFILE=infinity`就等于`LimitNOFILE=65536`，对于需求 10 万以上文件打开数的人，一定要自行设定。

综上所述，遇到`Too many open files`时的解决方法：

1. `ulimit -n`
2. `supervisor`：`minfds`
3. `systemd`：`LimitNOFILE`
