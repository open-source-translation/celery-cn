# 签名：Signatures

你刚刚在 [calling]() 指南中学习了如何通过 `delay` 方法来调用任务，并且这也是你常用的，但有时你可能希望传递任务签名给别的进程使用，或者作为其他函数的参数。

[signature()]() 包装了任务调用的参数、关键词参数和执行选项，以便传递给函数，甚至可以序列化后通过网络进行传输。

* 你可以创建签名，例如：
  ```python
  >>> from celery import signature
  >>> signature('tasks.add', args=(2, 2), countdown=10)
  tasks.add(2, 2)
  ```
  这个任务签名具有两个参数： (2, 2)，以及 countdown 为10的执行选项。

* 或者你可以使用任务的签名方法来创建签名：

  ```python
  >>> add.signature((2, 2), countdown=10)
  tasks.add(2, 2)
  ```

* 也有创建签名的快捷方式：

  ```python
  >>> add.s(2, 2)
  tasks.add(2, 2)
  ```

* 签名也支持关键词参数：

  ```python
  >>> add.s(2, 2, debug=True)
  tasks.add(2, 2, debug=True)
  ```

* 任何签名对象都可以检查不同的字段：

  ```python
  >>> s = add.signature((2, 2), {'debug': True}, countdown=10)
  >>> s.args
  (2, 2)
  >>> s.kwargs
  {'debug': True}
  >>> s.options
  {'countdown': 10}
  ```

* 签名也支持 `delay` 和 `apply_async` 的“调用API”，包括直接调用（\_\_call\_\_）。

  调用签名，将直接在当前进程中执行任务：
  ```python
  >>> add(2, 2)
  4
  >>> add.s(2, 2)()
  4
  ```
  `delay` 是我们代替 `apply_async` 的快捷方式：
  ```
  >>> result = add.delay(2, 2)
  >>> result.get()
  4
  >>> result = add.s(2, 2).delay()
  >>> result.get()
  4
  ```
  `apply_async` 采用与 [app.Task.apply_async()]() 相同的调用参数。
  ```python
  >>> add.apply_async(args, kwargs, **options)
  >>> add.signature(args, kwargs, **options).apply_async()

  >>> add.apply_async((2, 2), countdown=1)
  >>> add.signature((2, 2), countdown=1).apply_async()
  ```

* 你不能通过 [s()]() 定义执行选项，但是可以通过 set 的链式调用解决：
  ```python
  >>> add.s(2, 2).set(countdown=1)
  proj.tasks.add(2, 2)
  ```

## 部分参数
使用签名，你可以在工作进程中执行任务：
```python
>>> add.s(2, 2).delay()
>>> add.s(2, 2).apply_async(countdown=1)
```

或者，你也可以直接在当前进程执行任务：
```python
>>> add.s(2, 2)()
4
```

可以指明 `delay`/`apply_async` 额外的参数、关键词参数或执行选项：

* 任何参数都将添加在签名参数前：

  ```python
  >>> partial = add.s(2)          # incomplete signature
  >>> partial.delay(4)            # 4 + 2
  >>> partial.apply_async((4,))  # same
  ```

* 被添加的关键词参数将会和签名中的关键词参数合并，新添加的关键词参数优先：

  ```python
  >>> s = add.s(2, 2, debug=False)
  >>> s.delay(debug=True)                    # -> add(2, 2, debug=True)
  >>> s.apply_async(kwargs={'debug': True})  # same
  ```

* 被添加的执行选项将会与签名中的执行选项合并，新添加的执行选项优先：

  ```python
  >>> s = add.signature((2, 2), countdown=10)
  >>> s.apply_async(countdown=1)  # countdown is now 1
  ```

你也可以对签名进行克隆：
```python
>>> s = add.s(2)
proj.tasks.add(2)

>>> s.clone(args=(4,), kwargs={'debug': True})
proj.tasks.add(4, 2, debug=True)
```

## 不变性(Immutability)

部分参数通常在回调中使用，父任务的结果将会作为参数传递给链接或chord的回调任务。有时你希望指明一个不需要参数的回调，这时你可以设置签名为不变的。

```python
>>> add.apply_async((2, 2), link=reset_buffers.signature(immutable=True))
```

快捷方式 `.si()` 也能创建不变签名：

```python
>>> add.apply_async((2, 2), link=reset_buffers.si())
```

对于不变签名，只有执行选项可以进行设置，并无法使用部分参数和关键词参数。

注意：在该教材中，我有时会使用前缀 ~ 来进行签名。你不应该在生产环境中使用这样的代码，这仅仅是在 Python Shell 中进行测试的关键方式。

```python
>>> ~sig

>>> # is the same as
>>> sig.delay().get()
```

## 回调

任务可以使用 `apply_async` 的 link 参数来添加回调。

```python
add.apply_async((2, 2), link=other_task.s())
```

只有当任务成功退出，回调函数才能被执行，并且将父任务的结果作为参数传递给回调的任务。

如前所述，新传递的参数将会添加在签名指定的参数前。

如果你有一个签名：

```python
>>> sig = add.s(10)
```

接着 `sig.delay(result)` 将变为：

```python
>>> add.apply_async(args=(result, 10))
```


现在，让我们调用 `add` 任务，并链接回调。
```python
>>> add.apply_async((2, 2), link=add.s(8))
```

正如预期，首先第一个任务将会计算 2 + 2，接着回调任务将会计算 4 + 8。
