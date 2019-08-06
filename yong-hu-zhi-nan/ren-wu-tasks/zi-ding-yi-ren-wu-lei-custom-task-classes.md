# 自定义任务类：Custom task classes

所有的任务都继承 `app.Task` 类，`run()` 方法为任务体。

例如：

```python
@app.task
def add(x, y):
    return x + y
```

在内部大概会是这样：

```python
class _AddTask(app.Task):

    def run(self, x, y):
        return x + y
add = app.tasks[_AddTask.name]
```

## 实例化

任务不是为每一个请求进行实例化，而是作为全局实例在任务注册表中进行注册。

每一个进程只调用一次 `__init__` 构造函数，任务类在语义上更接近 Actor。

如果你有一个任务

```python
from celery import Task

class NaiveAuthenticateServer(Task):

    def __init__(self):
        self.users = {'george': 'password'}

    def run(self, username, password):
        try:
            return self.users[username] == password
        except KeyError:
            return False
```

将每一个请求路由到同一个进程中，然后它将处于保持状态。

对于缓存资源也是很有用的，例如，缓存数据库连接的基本任务类:

```python
from celery import Task

class DatabaseTask(Task):
    _db = None

    @property
    def db(self):
        if self._db is None:
            self._db = Database.connect()
        return self._db
```

可以添加到以下任务中：

```python
@app.task(base=DatabaseTask)
def process_rows():
    for row in process_rows.db.table.all():
        process_row(row)
```

`process_rows` 任务中的 `db` 属性在每个进程中始终保持不变。

## Handlers

**after\_return**\(self, status, retval, task\_id, args, kwargs, einfo\)

任务返回后调用的处理程序

Parameters:

* status – 当前任务状态
* retval – 任务返回值/异常
* task\_id – 唯一的任务ID
* args – 返回任务的原始参数
* kwargs – 返回任务的原始关键字

Keyword Arguments:

* einfo – 异常信息实例，包含 traceback （有的情况下）

此处理程序的返回值将被忽略。

**on\_failure**\(self, exc, task\_id, args, kwargs, einfo\)

任务执行失败时，由职程（Worker）调用。

Parameters:

* exc – 任务引发的异常。
* task\_id – 执行失败任务的唯一 ID。
* args – 任务失败的原始参数。
* kwargs – 任务失败的原始关键字。

Keyword Arguments:

* einfo – 异常信息实例，包含 traceback （有的情况下）。

此处理程序的返回值将被忽略。

**on\_retry**\(self, exc, task\_id, args, kwargs, einfo\)

任务重试时，由职程（Worker）调用。

Parameters:

* exc – 发送给 `retry()` 函数的异常
* task\_id – 任务重试唯一 ID。
* args – 任务重试的原始参数。
* kwargs – 任务重试的原始关键字。 Keyword Arguments:
* einfo – 异常信息实例，包含 traceback （有的情况下）。

此处理程序的返回值将被忽略。

**on\_success**\(self, retval, task\_id, args, kwargs\)

任务重试时，由职程（Worker）调用。

Parameters:

* retval – 任务的返回值
* task\_id – 执行成功唯一 ID。
* args – 任务执行成功时的原始参数。
* kwargs – 任务执行成功时的原始关键字。

此处理程序的返回值将被忽略。

