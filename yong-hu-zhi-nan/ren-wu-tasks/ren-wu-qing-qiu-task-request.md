# 任务请求：Task Request

`app.Task.request` 包含与当前执行任务相关的信息和状态。

该请求定义了以下属性:

| 属性名称 | 说明 |
| :--- | :--- |
| **id** | 执行任务的唯一ID |
| **group** | 任务组的唯一ID（该任务是组成员的情况下） |
| **chord** | 此任务所属的和弦的惟一id\(如果该任务是标题的一部分\) |
| **correlation\_id** | 用于重复数据删除的自定义ID |
| **args** | 选项参数 |
| **kwargs** | 关键字参数 |
| **origin** | 发送任务的主机名 |
| **retries** | 任务重试次数，默认是从 0 开始的 |
| **is\_eager** | 如果任务是由客户端执行，并非职程（Worker）执行，设置 True |
| **eta** | 任务预计时间（如果已经设置的情况下），时间为 UTC 格式（取决于 enable\_utc 设置） |
| **expires** | 任务执行到期时间（如果已经设置的情况下），默认为 UTC 格式（取决于 enable\_utc 设置） |
| **hostname** | 执行任务的职程（Worker）实例的节点名 |
| **delivery\_info** | 添加附加传递消息，主要用于包含交付任务的交换和路由键的映射，`retry()` 主要用于重新讲任务下发到队列中，该 dict 中的键可用取决于使用的消息中间人（Broker）。 |
| **reply-to** | 回复的发送的队列名称（例如，与 RPC 结果后端一起使用） |
| **called\_directly** | 如果职程（Worker）未执行任务，则此标志设置为true |
| **timelimit** | 当前\(软、硬\)时间限制的元组\(如果有的话\) |
| **callbacks** | 如果此任务成功返回，将调用的签名列表 |
| **errback** | 如果此任务失败，将调用的签名列表 |
| **utc** | 设置为 true ，启用 UTC |
| _3.1 版本的新功能_ | \*\*\*\* |
| **headers** | 与任务消息一起发送的消息头的映射（可以为 _None_） |
| **reply\_to** | 回复的地址（队列名称） |
| **correlation\_id** | 一般与任务的ID相同，通常用于AMQP中跟踪回复的内容 |
| _**4.0 版本的新功能**_ |  |
| **root\_id** | 此任务所属工作流中的第一个任务的唯一ID（如果有） |
| **parent\_id** | 调用此任务的任务的惟一id（如果有） |
| **chain** | 反转形成链的任务列表（如果有）。列表中最后一个任务是当前任务执行成功之后的下一个任务。如果使用任务协议的第一个版本，则链任务将位于 _request.callbacks_ 中 |

## 案例

访问上下文访问信息的一个任务案例：

```python
@app.task(bind=True)def dump_context(self, x, y):    print('Executing task id {0.id}, args: {0.args!r} kwargs: {0.kwargs!r}'.format(            self.request))
```

_bind_ 参数表示该函数绑是一个绑定方法，可以通过访问任务类型实例中的属性和方法。

