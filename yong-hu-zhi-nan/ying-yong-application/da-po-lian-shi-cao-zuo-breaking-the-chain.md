# 打破链式操作：Breaking the chain

虽然可以依赖于当前设置的应用程序，但最佳做法是始终将应用程序实例传递给需要它的任何内容。 通常将这种做法称为 `app chain`，根据传递的应用程序创建一系列实例。 下面的这个实例实不可取的：

```python
from celery import current_app

class Scheduler(object):

    def run(self):
        app = current_app
```

应该将 `app` 作为参数进行传递：

```python
class Scheduler(object):

    def __init__(self, app):
        self.app = app
```

在celery内部实现中，使用 celery.app\_or\_default\(\) 函数使得模块级别的 API 也能正常使用：

```python
from celery.app import app_or_default

class Scheduler(object):
    def __init__(self, app=None):
        self.app = app_or_default(app)
```

在开发环境中，可以通过设置 CELERY\_TRACE\_APP 环境变量在应用实例链被打破时抛出一个异常：

```bash
$ CELERY_TRACE_APP=1 celery worker -l info
```

{% hint style="info" %}
## API 发展



从 Celery 诞生， 2009 年就是发生了很大的变化。 例如，可以在开始时调用任何可调用任务：

```bash
def hello(to):
    return 'hello {0}'.format(to)

>>> from celery.execute import apply_async

>>> apply_async(hello, ('world!',))
```

也可以创建 Task 类来进行配置，覆盖其它行为：

```python
from celery.task import Task
from celery.registry import tasks

class Hello(Task):
    queue = 'hipri'

    def run(self, to):
        return 'hello {0}'.format(to)
tasks.register(Hello)

>>> Hello.delay('world!')
```

后来，决定传递任意可调用的是一个反模式，因为它使得使用除pickle之外的序列化器非常困难，并且该功能在2.0中被删除，被任务装饰器取代：

```python
from celery.task import task

@task(queue='hipri')
def hello(to):
    return 'hello {0}'.format(to)
```
{% endhint %}

