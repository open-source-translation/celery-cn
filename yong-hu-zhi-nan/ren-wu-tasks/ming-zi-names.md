# 名字：Names

每一个任务都必须有一个唯一任务名称。 如果没有指定任务名称，装饰器会根据当前任务所在的模块以及任务函数的名称进行生成一个： 设置名称案例：

```python
>>> @app.task(name='sum-of-two-numbers')>>> def add(x, y):...     return x + y>>> add.name'sum-of-two-numbers'
```

最好的做法是将模块名称作为任务名称空间，如果有一个任务与另外一个模块中的任务名称相同，这样就不会发生的冲突。

```bash
>>> @app.task(name='tasks.add')>>> def add(x, y):...     return x + y
```

可也通过调用任务的 `.name` 属性来进行获取任务名称：

```bash
>>> add.name'tasks.add'
```

在案例中我们指定的任务名称为 `task.add`，如果该任务函数在 `task.py` 文件中时自动生成任务与我们指定的任务名称相同：

tasks.py：

{% tabs %}
{% tab title="tasks.py" %}
```python
@app.taskdef add(x, y):    return x + y
```
{% endtab %}
{% endtabs %}

```bash
>>> from tasks import add>>> add.name'tasks.add
```

## 自动命名和相对导入

相对导入和自动命名兼容不是很好，如果导入方式为相对导入吗，建议明确设置名称。

{% hint style="info" %}
## 绝对导入 

Python 2 可以通过在**每个模块**中的顶部添加：

```python
from __future__ import absolute_import
```

使用之后会强制使用绝对导入，使用相对名称的任务不会出现问题。 绝对导入时 Python 3 默认的导入方式，如果开发使用的是 Python 3 就不需要写入以上代码。
{% endhint %}

例如，如果客户端导入模块 `myapp.tasks` 为 `.tasks`，职程（Worker）需要将模块导入为 `myapp.tasks`，生成的名称不会匹配，可能会出现 NotRegistered 错误信息。

在使用 Django 和 `project.myapp-style` 在 `INSTALLED_APPS` 命名时也会出现这种情况：

```text
INSTALLED_APPS = ['project.myapp']
```

如果以project.myapp的名称安装应用程序，则任务模块将作为project.myapp.tasks导入，因此必须确保始终使用相同的名称导入任务：

```bash
>>> from project.myapp.tasks import mytask   # << GOOD>>> from myapp.tasks import mytask    # << BAD!!!
```

在第二个案例中由于导入的模块不同，职程（Worker）和客户端会导致任务的命名不同：

```python
>>> from project.myapp.tasks import mytask>>> mytask.name'project.myapp.tasks.mytask'>>> from myapp.tasks import mytask>>> mytask.name'myapp.tasks.mytask'
```

所以，必须在导入模块的方式一致 同样不应该使用 old-style 进行导入：

```python
from module import foo   # BAD!from proj.module import foo  # GOOD!
```

可以使用 new-style 进行导入：

```python
from .module import foo  # GOOD!
```

在不重构的代码的情况下，可以直接指定名称不依赖自动命名来进行使用：

```python
@task(name='proj.tasks.add')def add(x, y):    return x + y
```

## 更改自动命名

4.0版中的新功能。 在某些情况默认的自动命名很合适。在许多不同的模块中有很多任务：

```bash
project/       /__init__.py       /celery.py       /moduleA/               /__init__.py               /tasks.py       /moduleB/               /__init__.py               /tasks.py
```

如果使用自动命名，每一个人都会生成一个名词，例如 `moduleA.tasks.taskA`、`moduleA.tasks.taskB`、`moduleB.tasks.test`等。可以通过重写 `app.gen_task_name()` 进行修改默认的所有任务名称中的 `tasks` 。基于刚刚的例子，稍微修改\(celery.py\)：

```python
from celery import Celeryclass MyCelery(Celery):    def gen_task_name(self, name, module):        if module.endswith('.tasks'):            module = module[:-6]        return super(MyCelery, self).gen_task_name(name, module)app = MyCelery('main')
```

这样所有的任务名称都类似 `moduleA.taskA`、`moduleA.taskB`、`moduleB.test`。

{% hint style="warning" %}
## 警告

需要确保 `app.gen_task_name()` 是一个函数，同样的输出和输出都必须相同。
{% endhint %}

