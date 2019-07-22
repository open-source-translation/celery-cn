# 抽象任务：Abstract Tasks

使用 `task()` 装饰器创建的所有任务都将从应用程序继承 Task 类。 也可以通过 `base` 参数进行指定基类：

```python
@app.task(base=OtherTask):
def add(x, y):
    return x + y
```

创建自定义任务类，需要继承 `celery.Task`：

```python
from celery import Task

class DebugTask(Task):

    def __call__(self, *args, **kwargs):
        print('TASK STARTING: {0.name}[{0.request.id}]'.format(self))
        return super(DebugTask, self).__call__(*args, **kwargs)
```

tip 如果重写 `taks` 中的 `__call__` 函数，就必须要要调用 `super()` 函数，这样这样基本调用方法可以设置直接调用任务时使用的默认请求。 基类是比较特殊的，因为它还没有绑定到任何特定的应用程序。一旦任务绑定到应用程序，它将读取配置以设置默认值，等等。 要实现基类，需要使用 `app.taks()` 装饰器创建任务：

```python
@app.task(base=DebugTask)
def add(x, y):
    return x + y
```

也可以通过更改应用程序的 `app.task()` 属性来更改其默认基类：

```python
>>> from celery import Celery, Task

>>> app = Celery()

>>> class MyBaseTask(Task):
...    queue = 'hipri'

>>> app.Task = MyBaseTask
>>> app.Task
<unbound MyBaseTask>

>>> @app.task
... def add(x, y):
...     return x + y

>>> add
<@task: __main__.add>

>>> add.__class__.mro()
[<class add of <Celery __main__:0x1012b4410>>,
 <unbound MyBaseTask>,
 <unbound Task>,
 <type 'object'>]
```

