# 如何工作：How it works

下面是技术细节。这部分不是你需要知道的，但是你可能会感兴趣。

所有定义的任务都在注册表中。注册表中包含任务名称和它们的任务类。你可以自己查看这个注册表：

```text
>>> from proj.celery import app
>>> app.tasks
{'celery.chord_unlock':
    <@task: celery.chord_unlock>,
 'celery.backend_cleanup':
    <@task: celery.backend_cleanup>,
 'celery.chord':
    <@task: celery.chord>}
```

这是 `celery`内置的任务列表，注意：当导入定义的模块时任务才会被注册。

默认加载程序导入配置 `imports` 列出的所有模块。

`app.task()` 装饰器负责在应用任务注册表中注册你的任务。

当任务被发送时，没有实际函数的代码被发送，只是将要执行的任务名称。当工作者收到消息时，它会在任务注册表中查找任务名称并执行代码。

这意味着你的工作者和客户端应该有着相同软件的更新。这个缺点的替代解决方案还有待解决。



