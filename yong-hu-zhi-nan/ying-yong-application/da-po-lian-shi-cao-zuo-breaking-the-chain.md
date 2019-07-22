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

