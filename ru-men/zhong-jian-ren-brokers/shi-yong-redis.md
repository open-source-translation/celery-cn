# 使用Redis

## 安装

如果使用 Redis 作为中间人（Broker）必须要安装 Celery 的依赖库，您可以通过 celery\[redis\] 进行安装：

```bash
$ pip install -U "celery[redis]"
```

## 配置

Redis 的配置非常的简单，只需要配置 Redis 的 URL ：

```text
app.conf.broker_url = 'redis://localhost:6379/0'
```

URL 的格式为：

```text
redis://:password@hostname:port/db_number
```

URL 的所有配置都可以自定义配置的，默认使用的是 localhost 的 6379 端口中 0 数据库。（ Redis 默认有 16 个数据库）

可以通过 Uninx 套接字进行连接，URl 格式如下：

```text
redis+socket:///path/to/redis.sock
```

可以通过设置 virtual\_host参数添加到URL上进行指定使用时 Uninx 套接字连接的数据库编号：

```text
redis+socket:///path/to/redis.sock?virtual_host=db_number
```

Celery 也可以连接 Redis 哨兵也是非常简单的：

```text
app.conf.broker_url = 'sentinel://localhost:26379;sentinel://localhost:26380;sentinel://localhost:26381'
app.conf.broker_transport_options = {'master_name':'cluster1'}
```

### 可见性超时

可见性超时为将消息重新下发给另外一个程序之前等待确认的任务秒数。请注意查看下面的 [注意事项](shi-yong-redis.md#zhu-yi-shi-xiang)。

可以通过 `broker_transport_options` 选项进行修改：

```text
app.conf.broker_transport_options = {'visibility_timeout': 3600} # 一个小时
```

默认的可见性超时时间为1个小时。

### 结果

如果您想保存任务执行返回结果保存到Redis，您需要进行以下配置：

```text
app.conf.result_backend = 'redis://localhost:7379/0'
```

有关 Redis 保存结果的完整选项列表，请查阅 Redis后端配置。

如果您使用的是 Redis 哨兵默认是，则需要使用 `result_backend_transport_options` 进行指定  master\_name：

```text
app.conf.result_backend_transport_options = {'master_name': "mymaster"}
```

## 注意事项

### 广播前缀

默认情况下，所有的虚拟机都可以看到广播的消息。

您必须为消息进行设置前缀，以便它们由仅活动的虚拟机接收：

```text
app.conf.broker_transport_options = {'fanout_prefix': true}
```

注意：该选项仅是向后兼容的，老版本不支持。集群中所有的职程都必须要开启设置，否则无法进行通信。

该设置在将来以后的版本是默认配置，所以请尽早进行迁移。

### 广播模式

默认情况下， 职程（Worker）收到所有与任务相关的事件。

为了避免该情况发生，需要进行配置 `fanout_patterns` 广播模式，以便职程（Worker）只能订阅相关的事件：

```text
app.conf.broker_transport_options = {'fanout_patterns': true}
```

该设置在将来以后的版本是默认配置。

### 可见性超时

如果在“可见性超时“内没有完成任务，该任务会重新分配给另外一个职程（Worker）进行执行。

这可能会出现在预计时间超出可见性超时时间的问题，如果出现该问题，任务将重新循环执行。

因此您必须要增加可见性超时时间用于用于匹配最长的执行时间。

注意：Celery会在职程（Worker）关闭的重新分配消息，如果可见性超时时间过长在断电或者强制终止职程（Worker）的情况会“丢失“重新分配的任务。

定期执行任务不会被可见性超时影响，因为这是俩个不同的概念。

您可以通过配置同名的配置选项来扩增可见性超时时间：

```text
app.conf.broker_yransport_options = {'visibility_timeout': 432000}
```

对应的值必须为 int 类型。

### 驱逐Key

在某些情况下，Redis会根据（驱逐策略）进行驱逐一些key

可能会出现已经错误问题：

```text
InconsistencyError: Probably the key ('_kombu.binding.celery') has been
removed from the Redis database.
```

您可以在Redis服务器的 time\_out 参数设置为0进行避免key被驱逐。

