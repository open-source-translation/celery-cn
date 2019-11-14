# 应用：Application

Celery 在使用前必须要进行实例化，实例化之后被称为应用程序（简称 app）。 应用程序是线程安全的，可以配置多个不同配置、组件、任务的 Celery 程序在同一个进程中运行。 创建一个实例：

```bash
>>> from celery import Celery
>>> app = Celery()
>>> app
<Celery __main__:0x100469fd0>
```

最后一行显示的内容包含 app 类的 Celery 名称，当前主模块 \(**main**\) 以及对象的内存地址（0x100469fd0）。

