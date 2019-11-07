# 主名称：Main Name

其中主模块的名称是比较重要的。 使用 Celery 进行发送任务消息时，消息内容不会含有源代码信息，只有执行任务的名称。这与主机名在互联网上的工作类似：每一个职程（Worker）通过任务名称与实际的功能进行对应，这样的方式被称为 任务注册 。 每当定义任务时，该任务也将添加到本地注册表中：

```bash
>>> @app.task... def add(x, y):...     return x + y>>> add<@task: __main__.add>>>> add.name__main__.add>>> app.tasks['__main__.add']<@task: __main__.add>
```

在上面的实例中 `__main__`再次出现，这是因为 Celery 无法检测到该函数属于哪个模块时，自动会使用主模块的名称作为任务名称的开头。 这只是一组有限实例中的问题：

* 如果在其中定义任务的模块作为程序运行。
* 如果应用程序是在python shell（repl）中创建的。

例如，这里的 tasks 模块还用于用 `app.worker_main()` 启动一个职程（Worker）:

tasks.py

{% tabs %}
{% tab title="tasks.py" %}
```python
from celery import Celeryapp = Celery()@app.taskdef add(x, y): return x + yif __name__ == '__main__':    app.worker_main()
```
{% endtab %}
{% endtabs %}

当模块执行时，任务名以 `__main__` 开头，被另外一个模块导入时，例如调用一个任务时，任务的名称将以 `tasks` （模块真实的名称）进行开头命名：

```bash
>>> from tasks import add>>> add.nametasks.add
```

也可以设置主模块的名称：

```bash
>>> app = Celery('tasks')>>> app.main'tasks'>>> @app.task... def add(x, y):...     return x + y>>> add.nametasks.add
```

{% hint style="info" %}
## See one see （可以看一看）

[任务：Tasks](https://app.gitbook.com/@open-source-translation/s/celery/~/edit/drafts/-LkHOQZTEvY7SK2wkQc-/yong-hu-zhi-nan/ren-wu-tasks) 章节中 Names。
{% endhint %}

