# Celery 4.4.0的新功能：What’s new in Celery 4.4 \(Cliffs\)

作者: Asif Saif Uddin \(auvipy at gmail.com\)

`Celery` 是一个用于处理大批量的消息的简单，灵活并且可靠的分布式程序框架，并且提供了以 `Python` 编写的用于维护一个分布式系统必要的工具。

它是一个专注与实时处理的任务队列，并且也支持定时任务。

`Celery` 有一个拥有庞大且多样的用户和志愿者的社区, 你可以通过 [IRC](http://docs.celeryproject.org/en/stable/getting-started/resources.html#irc-channel) 加入我们或者加入 [我们的邮件列表](http://docs.celeryproject.org/en/stable/getting-started/resources.html#mailing-list)

要了解更多关于 `Celery` 的信息，你可以去阅读 [介绍](http://docs.celeryproject.org/en/stable/getting-started/introduction.html#intro)

虽然此版本向后兼容之前发布的版本，但阅读下方的章节也是很重要的。

当前版本官方支持 CPython 2.7，3.5，3.6，3.7 以及 3.8，并且也支持 PyPY2 和 PyPy3。

### 序

4.4.0 版本继续改进我们的工作，以为你提供 `Python` 语言下最好的任务执行平台。

此版本的代号为 [Cliffs](https://www.youtube.com/watch?v=i524g6JMkwI)， 这是我最喜欢的曲目。

此版本主要针对于 `bug` 修复以及开发人员的易用性改进。减少了很多长期存在的`bug`, 易用性问题，文档问题和一些较小的改进问题 用来改善整体开发人员的体验。

`Celery 4.4` 是第一个支持 Python 3.8 和 PyPy36-7.2 的版本.

在我们开始对 `Celery 5`\(我们下一代的任务执行平台\)的工作时，在 Celery 5 的稳定版本出现之前，至少会有另一个 4.x 版本，并且会获得至少一年的支持, 具体取决于社区的需求和支持。

我们也针对减少贡献的冲突更新了贡献工具。

_— Asif Saif Uddin_

#### **贡献者墙**

> **注意:**
>
>  这里是从 `git` 的提交历史中自动生成的，因此遗憾的是并不包含那些为更重要的事情\(如回答邮件列表里的问题\)提供了帮助的人。

### 从 Celery 4.3 升级

请阅读下方的重要提示，因为其中包含了一些重大更改。

### 重要提示

#### **支持的 Python 版本**

支持的 Python 版本有：

* CPython 2.7
* CPython 3.5
* CPython 3.6
* CPython 3.7
* CPython 3.8
* PyPy2.7 7.2 \(pypy2\)
* PyPy3.5 7.1 \(pypy3\)
* PyPy3.6 7.2 \(pypy3\)

#### **停止支持 Python 3.4**

Celery 现在要求 Python 2.7 或者 Python 3.5 及其以上版本。

Python 3.4 已经在 2019 年 5 月到达产品结束时间。为了集中精力我们在这个版本停止了对 Python 3.4 的支持。

如果你仍然需要在 Python 3.4 上运行 Celery。你可以仍然使用 Celery 4.3 版本。但是我们建议你升级到支持的 Python 版本，因为 Python 3.4 将不会再应用新的安全补丁。

#### **Kombu**

从此版本开始，最低要求的版本是 Kombu 4.6.6

#### **Billiard**

从此版本开始，最低要求的版本是 Billiard 3.6.1

#### **Redis 消息中间件**

由于redis-py的早期版本中存在多个导致Celery出现问题的 bug，我们不得不将最低要求的版本提高到3.3.0。

#### **Redis 结果存储**

由于redis-py的早期版本中存在多个导致Celery出现问题的 bug，我们不得不将最低要求的版本提高到3.3.0。

#### **DynamoDB 结果存储**

DynamoDB 结果存储已经得到了 TTL 的支持。将 boto3 的版本提升到 1.9.178，这是第一个对 DynamoDB 支持 TTL 的版本。

**S3 结果存储**

为了追赶当前 AWS API 的更改，将 boto3 的版本提升到了 1.9.125。

#### **SQS 消息中间件**

为了追赶当前 AWS API 的更改，将 boto3 的版本提升到了 1.9.125。

### **配置**

`CELERY_TASK_RESULT_EXPIRES` 配置项已经被替换为 `CELERY_RESULT_EXPIRES`。

### 新的东西

#### **任务池**

**线程任务池**

我们通过 `concurrent.futures.ThreadPoolExecutor` 引入了线程任务池**。**

这之前的线程任务池是实验性的功能，此外它还基于已经过时的软件包 [threadpool](https://pypi.org/project/threadpool/)。

你可以通过设置 [worker\_pool](http://docs.celeryproject.org/en/stable/userguide/configuration.html#std:setting-worker_pool) 为 `threads` 或者在 celery 职称命令行里传递 `–pool threads` 参数来使用新的线程任务池。

#### **结果存储**

**ElasticSearch 结果存储**

**HTTP 基础认证支持**

如今在使用 `ElasticSearch` 结果存储时，你可以在 URL 中提供用户名和密码来使用 HTTP 基础认证。

在之前，认证参数被忽略并且只支持无需授权的请求。

**MongoDB 结果存储**

**认证源和认证方法的支持**

现在你可以通过 URL 选项为 MongoDB 指定 `authSource` 和 `authMethod`。下方的 URL 就是这样做的：

```text
mongodb://user:password@example.com/?authSource=the_database&authMechanism=SCRAM-SHA-256
```

有关各种选项的具体信息，请参阅 [文档](https://api.mongodb.com/python/current/examples/authentication.html)。

#### 任务

**任务类的定义现在可以拥有重试的属性**

你可以在基于类的任务中使用 `autoretry_for`，`retry_kwargs`，`retry_backoff_max`和 `retry_jitter`属性：

```text
class BaseTaskWithRetry(Task):
  autoretry_for = (TypeError,)
  retry_kwargs = {'max_retries': 5}
  retry_backoff = True
  retry_backoff_max = 700
  retry_jitter = False
```

#### **Canvas**

**紧急任务替换**

现在你可以在急切运行的任务上调用 `self.replace()`。他们将在异步运行同样正常的工作。

**链式组**

链式组不再只产生单个组

如下方式之前用来将两个组合并为一个。 现在他们将正确的一个接一个的执行：

```text
>>> result = group(add.si(1, 2), add.si(1, 2)) | group(tsum.s(), tsum.s()).delay()
>>> result.get()
[6, 6]
```



  


