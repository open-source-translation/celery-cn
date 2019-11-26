# 签名：Signatures

你刚刚在 [calling]() 指南中学习了如何通过 `delay` 方法来调用任务，并且这也是你经常需要的，但有时你可能希望传递任务调用的签名给别的进程，或者作为其他函数的参数。

[signature()]() 包装了任务调用的参数、关键词参数和执行选项，以便传递给函数，甚至可以序列化通过网络进行传输。

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

指明 `delay`/`apply_async` 额外的参数、关键词参数或可执行选项，可以创建部分参数。

* 任何参数都将在签名的参数前被添加：

  ```python
  >>> partial = add.s(2)          # incomplete signature
  >>> partial.delay(4)            # 4 + 2
  >>> partial.apply_async((4,))  # same
  ```

* Any keyword arguments added will be merged with the kwargs in the signature, with the new keyword arguments taking precedence:

  ```python
  >>> s = add.s(2, 2)
  >>> s.delay(debug=True)                    # -> add(2, 2, debug=True)
  >>> s.apply_async(kwargs={'debug': True})  # same
  ```

* Any options added will be merged with the options in the signature, with the new options taking precedence:

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

## Immutability

Partials are meant to be used with callbacks, any tasks linked, or chord callbacks will be applied with the result of the parent task. Sometimes you want to specify a callback that doesn’t take additional arguments, and in that case you can set the signature to be immutable:

```python
>>> add.apply_async((2, 2), link=reset_buffers.signature(immutable=True))
```

The .si() shortcut can also be used to create immutable signatures:

```python
>>> add.apply_async((2, 2), link=reset_buffers.si())
```

Only the execution options can be set when a signature is immutable, so it’s not possible to call the signature with partial args/kwargs.

注意：In this tutorial I sometimes use the prefix operator ~ to signatures. You probably shouldn’t use it in your production code, but it’s a handy shortcut when experimenting in the Python shell:

```python
>>> ~sig

>>> # is the same as
>>> sig.delay().get()
```

## 回调

Callbacks can be added to any task using the link argument to apply_async:

```python
add.apply_async((2, 2), link=other_task.s())
```

The callback will only be applied if the task exited successfully, and it will be applied with the return value of the parent task as argument.

As I mentioned earlier, any arguments you add to a signature, will be prepended to the arguments specified by the signature itself!

If you have the signature:

```python
>>> sig = add.s(10)
```

then sig.delay(result) becomes:
```python
>>> add.apply_async(args=(result, 10))
```

Now let’s call our add task with a callback using partial arguments:

```python
>>> add.apply_async((2, 2), link=add.s(8))
```

As expected this will first launch one task calculating 2 + 2, then another task calculating 4 + 8.
