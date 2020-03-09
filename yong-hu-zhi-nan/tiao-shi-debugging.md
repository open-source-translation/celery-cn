# 调试：Debugging

## 远程调试任务 \(使用 pdb\)

### 基本用法

[`celery.contrib.rdb`](http://docs.celeryproject.org/en/latest/reference/celery.contrib.rdb.html#module-celery.contrib.rdb) 是 [`pdb`](https://docs.python.org/dev/library/pdb.html#module-pdb) 的一个扩展版本，可以对没有终端访问权限的进程进行远程调试。

使用示例：

```text
from celery import task
from celery.contrib import rdb

@task()
def add(x, y):
    result = x + y
    rdb.set_trace()  # <- set break-point
    return result
```

[`set_trace()`](http://docs.celeryproject.org/en/latest/reference/celery.contrib.rdb.html#celery.contrib.rdb.set_trace) 在当前位置设置了一个断点，并创建了一个可以通过 telnet 连接的管道用以调试你的任务。

调试器可能同时由多个进程启动，所以调试器将从基础端口\(默认情况下为6900\)开始搜索可用的端口，而不是使用一个固定的端口。基础端口可以通过环境变量 [`CELERY_RDB_PORT`](http://docs.celeryproject.org/en/latest/reference/celery.contrib.rdb.html#envvar-CELERY_RDB_PORT) 来进行修改。

默认情况下调试器将只对本机可用，你可以通过 [`CELERY_RDB_HOST`](http://docs.celeryproject.org/en/latest/reference/celery.contrib.rdb.html#envvar-CELERY_RDB_HOST) 来设置允许从外界访问调试器。

当职程执行到你指定的断点处时，将会打印如下日志信息：

```text
[INFO/MainProcess] Received task:
    tasks.add[d7261c71-4962-47e5-b342-2448bedd20e8]
[WARNING/PoolWorker-1] Remote Debugger:6900:
    Please telnet 127.0.0.1 6900.  Type `exit` in session to continue.
[2011-01-18 14:25:44,119: WARNING/PoolWorker-1] Remote Debugger:6900:
    Waiting for client...
```

如果你使用 telnet 连接了指定的端口，将会显示一个 `pdb` 的 shell：

```text
$ telnet localhost 6900
Connected to localhost.
Escape character is '^]'.
> /opt/devel/demoapp/tasks.py(128)add()
-> return result
(Pdb)
```

输入 help 可以获取可用的命令列表，如果您之前并没有用过 `pdb` ，最好先阅读一下文档 [Python Debugger Manual](http://docs.python.org/library/pdb.html) 。

为了演示，我们将读取 `result` 变量的值，对其进行修改之后继续执行任务：

```text
(Pdb) result
4
(Pdb) result = 'hello from rdb'
(Pdb) continue
Connection closed by foreign host.
```

我们修改的结果将在职程的日志呈现：

```text
[2011-01-18 14:35:36,599: INFO/MainProcess] Task
    tasks.add[d7261c71-4962-47e5-b342-2448bedd20e8] succeeded
    in 61.481s: 'hello from rdb'
```

### 提示

#### 启用断点信号

如果设置了环境变量 CELERY\_RDBSIG ，每当收到 SIGUSR2 信号时，就会打开一个 rdb 的实例。主进程和工作进程都是这种情况。

启动职程的示例：

```text
$ CELERY_RDBSIG=1 celery worker -l info
```

你可以通过执行如下命令为任何工作职程启动一个 `rdb` 的会话：

```text
$ kill -USR2 <pid>
```

