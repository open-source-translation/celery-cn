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

