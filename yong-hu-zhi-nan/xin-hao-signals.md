# 信号：Signals

当某些动作在应用中的其他位置触发时，信号允许解耦的应用接收到通知。

Celery 附带了很多信号，您的应用可以嵌入这些信号来增加某些动作的额外行为。

## 基本

一些事件触发信号，你可以连接到这些信号以在它们被触发时执行一些操作。

连接到 [`after_task_publish`](https://docs.celeryproject.org/en/stable/userguide/signals.html#std:signal-after_task_publish) 信号的示例：

```python
from celery.signals import after_task_publish

@after_task_publish.connect
def task_sent_handler(sender=None, headers=None, body=None, **kwargs):
    # 任务的信息位于任务信息的标题中
    # 使用第二版的任务协议.
    info = headers if 'task' in headers else body
    print('after_task_publish for task id {info[id]}'.format(
        info=info,
    ))
```

一些信号也有一个你可以进行过滤的发送者。比如 [`after_task_publish`](https://docs.celeryproject.org/en/stable/userguide/signals.html#std:signal-after_task_publish)  信号使用任务的名称作为发送者，所以通过提供发送者的参数进行连接，你可以在每次名称为 `proj.tasks.add` 的任务被发布时连接要被调用的处理函数：

```python
@after_task_publish.connect(sender='proj.tasks.add')
def task_sent_handler(sender=None, headers=None, body=None, **kwargs):
    # information about task are located in headers for task messages
    # using the task protocol version 2.
    info = headers if 'task' in headers else body
    print('after_task_publish for task id {info[id]}'.format(
        info=info,
    ))
```

信号使用与 `django.core.dispatch` 相同的实现，所以在默认情况下，其他的关键字参数\(如 singal\) 将被传递到所有的信号处理函数里。

信号处理的最佳实践是接受任意的关键字参数\(如, \*\*kwargs\)。这种方式使得新的 `Celery` 版本可以在不影响用户代码的情况下添加可选参数。

## 信号

### 任务信号

#### **before\_task\_publish**

在 3.1 版本引入

在任务发布之前派发。注意，这是在发送任务的过程中执行的。

发送者时被发布的任务的名称。

提供的参数:

* body

  任务消息体

  这是一个包含任务消息字段的映射，有关定义的可能字段的参考，请参阅 [Version 2](https://docs.celeryproject.org/en/stable/internals/protocol.html#message-protocol-task-v2)  和 [Version 1](https://docs.celeryproject.org/en/stable/internals/protocol.html#message-protocol-task-v1)。

* 
