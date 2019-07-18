# Celery 进阶使用

[`Celery 初次使用`](celery-chu-ci-shi-yong.md)章节简单的说明了一下 Celery 的基本使用，本章节将更详细的介绍和使用 Celery，其中包含在自己的应用程序中和库中使用 Celery。

本章节结尾记录有 Celery 的所有功能和最佳实战，建议继续阅读“用户指南”。

##  在应用程序使用

### 我们的项目

项目的结构：

```bash
proj/__init__.py
    /celery.py
    /tasks.py
```

proj/celery.py

{% code-tabs %}
{% code-tabs-item title="proj/celery.py" %}
```python
from __future__ import absolute_import, unicode_literals
from celery import Celery

app = Celery('proj',
             broker='amqp://',
             backend='amqp://',
             include=['proj.tasks'])

# Optional configuration, see the application user guide.
app.conf.update(
    result_expires=3600,
)

if __name__ == '__main__':
    app.start()
```
{% endcode-tabs-item %}
{% endcode-tabs %}

在此程序中，创建了 Celery 实例（也称 `app`），如果需要使用 Celery，导入即可。

* 该 broker 参数为指定的中间人（Broker）URL。

  有关更多信息，请查阅[`选择中间人`](celery-chu-ci-shi-yong.md#xuan-ze-zhong-jian-ren-broker) 。

* 该 backend 参数为指定的结果后端 URL。

  它主要用于跟踪任务的状态的信息，默认情况下禁用结果后端，在此实例中已经开启了该功能，主要便于后续检索，可能在会在程序中使用不同的结果后端。每一个后端都有不同的优点和缺点，如果不需要结果，最好禁用。也可以通过设置 `@task(ignore_result=True)` 参数，针对单个任务禁用。

  有关更多详细材料，请参阅[`保存结果`](celery-chu-ci-shi-yong.md#bao-cun-jie-guo) 。

* 该 include 参数是程序启动时倒入的模块列表，可以该处添加任务模块，便于职程能够对应相应的任务。

proj/tasks.py

{% code-tabs %}
{% code-tabs-item title="proj/tasks.py" %}
```python
from __future__ import absolute_import, unicode_literals
from .celery import app

@app.task
def add(x, y):
    return x + y

@app.task
def mul(x, y):
    return x * y

@app.task
def xsum(numbers):
    return sum(numbers)
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 运行职程（Worker）

Celery 程序可以用于启动职程（Worker）：

```bash
$ celery -A proj worker -l info
```

当职程（Worker）开始运行时，可以看到一部分日志消息：

```bash
-------------- celery@halcyon.local v4.0 (latentcall)
---- **** -----
--- * ***  * -- [Configuration]
-- * - **** --- . broker:      amqp://guest@localhost:5672//
- ** ---------- . app:         __main__:0x1012d8590
- ** ---------- . concurrency: 8 (processes)
- ** ---------- . events:      OFF (enable -E to monitor this worker)
- ** ----------
- *** --- * --- [Queues]
-- ******* ---- . celery:      exchange:celery(direct) binding:celery
--- ***** -----

[2012-06-08 16:23:51,078: WARNING/MainProcess] celery@halcyon.local has started.
```

## 程序调用

