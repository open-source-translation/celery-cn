# 基于 Eventlet 的并发

## 介绍

[Eventlet](http://eventlet.net/) 主页将它描述为 Python 的并发网络库，允许你更改你的代码的运行方式，而不是编写方式。

* 它使用 [epoll\(4\)](http://linux.die.net/man/4/epoll)  或 [libevent](http://monkey.org/~provos/libevent/) 来实现 [高度可扩展的非阻塞的 I/O](https://en.wikipedia.org/wiki/Asynchronous_I/O#Select.28.2Fpoll.29_loops)。
* [协程](https://en.wikipedia.org/wiki/Coroutine) 确保开发人员使用类似线程的阻塞式编程，但提供非阻塞式 I/O 的好处。
* 事件的下发是隐式的：意味着您可以轻松的在 Python 解释器中使用 Eventlet，也可以将其作为大型应用程序的一部分。

`Celery` 支持 `Eventlet` 作为可选执行池的实现，在某些情况下要优于 `prefork` 。但是你需要确保一项任务不会太长时间阻塞事件循环。一般来说，与 CPU 绑定的操作不适用与 Eventlet。并且还要注意的是，一些第三方的库，通常指带有C扩展的，由于无法使用猴子补丁，因此不能从使用 `Eventet` 中获得益处。如果你无法确定，可以参考它们的官方文档。如 `pylibmc` 不允许于 `Eventlet` 一起使 用，但是 `psycopg2` 可以，虽然它们都是带有 C 扩展的库。

`prefork` 池是利用多进程，但是数量受限于每个 `CPU` 只能有几个进程。使用 `Eventlet`，您可以有效地产生数百或者数千个绿色线程。在一个动态中转系统的非正式测试中，`Eventlet` 池可以每秒获取并处理数百个动态，而 `prefork` 池处理 100 条动态花费了14秒之多。但请注意，这是 异步 I/O 的优势所在\(异步的HTTP请求\)。您也许需要将 `Eventlet` 和 `prefork` 职程搭配使用，并根据兼容性或者最适合处理的角度来路由任务。

## 启用 Eventlet

你可以使用 `Celery` 的职程参数 -P 来启用 `Eventlet`。

```text
$ celery -A proj worker -P eventlet -c 1000
```

## 示例

有关使用 Eventlet  支持的示例，请参阅 `Celery` 发行版本的 [Eventlet 示例](https://github.com/celery/celery/tree/master/examples/eventlet) 文件夹

