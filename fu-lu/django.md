# Django

celery集合django使用

注意 以前版本的Celery需要一个单独的库才能与Django一起使用，但是从3.1开始，情况不再如此。现在就已支持Django，因此本文档仅包含集成Celery和Django的基本方法。你将使用与非Django用户相同的API，因此建议先阅读"[使用Celery进行入门](https://www.celerycn.io/ru-men/celery-chu-ci-shi-yong)"教程，然后再返回本教程。当有可用的示例时，可以继续阅读["下一步"](https://docs.celeryproject.org/en/v4.3.0/getting-started/next-steps.html#next-steps)指南。

注意Celery 4.0支持Django 1.8和更高版本。对于Django 1.8之前的版本，请使用Celery 3.1。

要将Celery与Django项目一起使用，必须首先定义Celery库的实例(称为 "app") 

如果您拥有当前主流的Django项目结构，例如：
```
- proj/
  - manage.py
  - proj/
    - __init__.py
    - settings.py
    - urls.py
```
那么建议的方法是创建一个新的proj/proj/celery.py模块，该模块定义Celery实例

**file**:	proj/proj/celery.py
```python
from __future__ import absolute_import, unicode_literals
import os
from celery import Celery

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'proj.settings')

app = Celery('proj')

# Using a string here means the worker doesn't have to serialize
# the configuration object to child processes.
# - namespace='CELERY' means all celery-related configuration keys
#   should have a `CELERY_` prefix.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django app configs.
app.autodiscover_tasks()


@app.task(bind=True)
def debug_task(self):
    print('Request: {0!r}'.format(self.request))
```
然后，需要将此程序导入到`proj/proj/__ init__.py`模块中。这样可以确保在Django启动时加载应用程序，以便@shared_task装饰器（稍后提及）将使用该应用程序：

**file**:`proj/proj/__init__.py:`
```
from __future__ import absolute_import, unicode_literals

# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app

__all__ = ('celery_app',)
```
请注意，此示例项目结构适用于较大的项目，对于简单项目，可以使用一个包含的模块来定义应用程序和任务，例如"[使用Celery进行入门](https://www.celerycn.io/ru-men/celery-chu-ci-shi-yong)"教程。

让我们分解一下第一个模块中发生的情况，首先我们从`__freture__`导入`absolute_import`导入`unicode_literals`，以使我们的celery.py模块不会与库冲突：
```
from __future__ import absolute_import
```
然后，为celery命令行程序设置默认的[`DJANGO_SETTINGS_MODULE`](https://django.readthedocs.io/en/latest/topics/settings.html#envvar-DJANGO_SETTINGS_MODULE)环境变量：
```
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'proj.settings')
```
不需要此行，但可以避免始终将设置模块传递到celery程序中。它必须始终在创建应用程序实例之前出现，我们接下来要做的是：
```
app = Celery('proj')
```
这是我们的库实例，可以有很多实例，但是使用Django时可能没有理由。
我们还将Django设置模块添加为Celery的配置源。这意味着不必使用多个配置文件，而直接从Django设置中配置Celery；但也可以根据需要将它们分开。
```
app.config_from_object('django.conf:settings', namespace='CELERY')
```
大写名称空间意味着所有Celery配置选项必须以大写而不是小写指定，并以`CELERY_`开头，因此例如[`task_always_eager`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-task_always_eager)设置变为`CELERY_TASK_ALWAYS_EAGER`，[`broker_url`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-broker_url)设置变为`CELERY_BROKER_URL`。

可以直接传递设置对象，但是使用字符串更好，因为这样一来，工作人员就不必序列化该对象。 `CELERY_`名称空间也是可选的，但建议使用（以防止与其他Django设置重叠）。

接下来，可重用应用程序的常见做法是在单独的task.py模块中定义所有任务，而Celery可以自动发现这些模块：
```
app.autodiscover_tasks()
```
在上述行中，Celery将按照task.py约定自动发现所有已安装应用程序中的任务：
```
- app1/
    - tasks.py
    - models.py
- app2/
    - tasks.py
    - models.py
```
这样，不必手动将各个模块添加到[`CELERY_IMPORTS`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-imports)设置中。
最后，debug_task示例是一个转储其自己的请求信息的任务。这是使用Celery 3.1中引入的新的`bind = True`任务选项来轻松引用当前任务实例。

### 使用@shared_task装饰器 
编写的任务可能会存在于可重用应用程序中，并且可重用应用程序不能依赖于项目本身，因此不能直接导入应用程序实例。
@shared_task装饰器可让您创建任务而无需任何具体的应用程序实例：
**file**: demoapp/tasks.py:
```
# Create your tasks here
from __future__ import absolute_import, unicode_literals
from celery import shared_task


@shared_task
def add(x, y):
    return x + y


@shared_task
def mul(x, y):
    return x * y


@shared_task
def xsum(numbers):
    return sum(numbers)
```
ps: 可以在以下位置找到Django示例项目的完整源代码：
https://github.com/celery/celery/tree/master/examples/django/

相对倒入
必须在导入任务模块的方式上保持一致。例如，如果您在`INSTALLED_APPS`中有project.app，则还必须从project.app中导入任务，否则任务的名称最终将不同。
[Automatic naming and relative imports](https://docs.celeryproject.org/en/v4.3.0/userguide/tasks.html#task-naming-relative-imports)

扩展
django-celery-results - Using the Django ORM/Cache as a result backend
[django-celery-results](https://pypi.org/project/django-celery-results/)扩展使用Django ORM或Django Cache框架提供results后端。

要在您的项目中使用此功能，您需要遵循以下步骤
1 Install the django-celery-results library:
```
$ pip install django-celery-results
```
2 在Django项目的settings.py中将django_celery_results添加到INSTALLED_APPS：
```
INSTALLED_APPS = (
    ...,
    'django_celery_results',
)
```
3 通过执行数据库迁移来创建Celery数据库表：
```
$ python manage.py migrate celery_results
```
4 配置Celery以使用django-celery-results后端。
假设您使用Django的settings.py来配置Celery，请添加以下设置：
```
CELERY_RESULT_BACKEND = 'django-db'
```
对于缓存后端，可以使用：
```
CELERY_CACHE_BACKEND = 'django-cache'
```

[django-celery-beat](https://docs.celeryproject.org/en/v4.3.0/userguide/periodic-tasks.html#beat-custom-schedulers) - Database-backed Periodic Tasks with Admin interface.

### 启动Worker
在生产环境中，您需要在后台将守护进程作为守护程序运行-请参见[守护进程](https://www.celerycn.io/yong-hu-zhi-nan/shou-hu-jin-cheng-daemonization)-但对于测试和开发，能够使用`celery worker manage`命令启动一个守护程序实例非常有用，就像使用Django的`manage.py runserver`:
```
$ celery -A proj worker -l info
```
有关可用的命令行选项的完整列表，请使用help命令：
```
$ celery help
```
接下来看点啥
如果您想了解更多信息，请继续阅读[“下一步”](https://docs.celeryproject.org/en/v4.3.0/getting-started/next-steps.html#next-steps)教程，然后您可以阅读[《用户指南》](https://docs.celeryproject.org/en/v4.3.0/userguide/index.html#guide)。