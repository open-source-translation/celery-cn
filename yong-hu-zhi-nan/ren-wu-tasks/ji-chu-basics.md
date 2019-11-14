# 基础：Basics

可以使用 `task()` 装饰器轻松的创建任何可被调用的任务：

```python
from .models import User

@app.task
def create_user(username, password):
    User.objects.create(username=username, password=password)
```

也可以通过 `task()`装饰器针对任务设置 `选项参数`：

```python
@app.task(serializer='json')
def create_user(username, password):
    User.objects.create(username=username, password=password)
```

{% hint style="info" %}
## 多个装饰器

如果有多个装饰器进行组合使用时，必须确保 `task()` 装饰器被放置在首位：

```python
@app.task
@decorator2
@decorator1
def add(x, y):
    return x + y
```
{% endhint %}

{% hint style="info" %}
## 如何导入装饰器？什么是“应用程序”？

任务装饰器在 Celery 应用程序实例中使用，如果不清楚是什么，建议阅读 [`Celery 初次使用`](../../ru-men/celery-chu-ci-shi-yong.md)。 如果使用的是 Django （参阅 Django 的 `初次使用`），如果你是一个库的作者，你可能使用想要使用 `shared_task()` 装饰器：

```python
from celery import shared_task

@shared_task
def add(x, y):
    return x + y
```
{% endhint %}

## 绑定的任务

被绑定的任务意味着任务的第一个参数总是任务实例（self），就像Python绑定方法一样:

```python
logger = get_task_logger(__name__)

@task(bind=True)
def add(self, x, y):
    logger.info(self.request.id)
```

对于重试（使用 `app.Task.retry()`）、访问当前任务的信息以及添加到自定义任务基类的任何附加功能，都需要绑带任务。

## 任务继承

装饰器的 `base` 参数可以指定继承的任务基类：

```python
import celery

class MyTask(celery.Task):

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        print('{0!r} failed: {1!r}'.format(task_id, exc))

@task(base=MyTask)
def add(x, y):
    raise KeyError()
```

