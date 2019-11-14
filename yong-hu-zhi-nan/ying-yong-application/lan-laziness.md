# 懒加载：Laziness

程序是`懒加载`的，在没有实际使用的情况下，是不会进行加载的。 创建一个 Celery 程序的流程如下：

* 创建用于事件的逻辑时钟实例
* 创建任务注册表
* 将自身设置为当前应用程序（如果禁用 `set_as_current` 参数则不会）
* 调用 `app.on_init()` 回调函数\(默认情况下不执行任何操作\)

  `app.task()` 装饰器不会在定义任务时创建任务，创建任务通常在使用该任务或应用程序完成后进行创建。

该实例会显示，在使用任务或访问属性之前，是如何创建任务的：

```bash
>>> @app.task
>>> def add(x, y):
...    return x + y

>>> type(add)
<class 'celery.local.PromiseProxy'>

>>> add.__evaluated__()
False

>>> add        # <-- causes repr(add) to happen
<@task: __main__.add>

>>> add.__evaluated__()
True
```

应用程序的终结可以通过调用 `app.finalize()` 显式执行，也可以通过访问 `app.tasks` 属性隐式执行。 完成创建对象将：

* 复制必须在应用之间共享的任务

    默认情况下共享任务，如果装饰器的 `shared` 参数，该任务为私有任务。

* 评估所有待处理的任务装饰器
* 确保当前所有的人呢我都已经绑定到当前应用程序

    任务绑带到应用程序，便于从配置中获取配置信息。

{% hint style="info" %}
## default app

Celery 不只是应用程序，过去只有基于模块的 API吗，为了向后兼容，在 Celery 5.0 发布之前，旧的 API 会继续保留。 Celery 会创建一个特殊的应用程序（默认应用程序），如果实例化自定义的应用程序，会直接使用 Celery 创建的默认程序。 celery.task 模块主要用于兼容旧的 API ，如果自定义创建应用创建，不建议使用 celery.task。 老的 Task 基类中启用了许多兼容性功能，其中有些功能可能与最新的不兼容，使用方法如下：

from celery.task import Task \# &lt;&lt; OLD Task base class.

 from celery import Task \# &lt;&lt; NEW base class.

建议进行升级为新的基类。
{% endhint %}

