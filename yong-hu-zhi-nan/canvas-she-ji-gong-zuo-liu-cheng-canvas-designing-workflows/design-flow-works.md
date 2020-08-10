Canvas: 设计工作流
### 签名 Signatures
2.0版的新功能。
您刚刚在调用指南中学习了如何使用任务延迟方法来[调用](https://docs.celeryproject.org/en/v4.3.0/userguide/calling.html#guide-calling)任务，这通常是您所需要的，但有时您可能希望将任务调用的签名传递给另一个进程或作为另一个函数的参数。

[`signature()`](https://docs.celeryproject.org/en/v4.3.0/reference/celery.html#celery.signature)装饰单个任务调用的参数，关键字参数和执行选项，以便可以将其传递给函数，甚至可以通过**wire**进行序列化和发送

```
>>> from celery import signature
>>> signature('tasks.add', args=(2, 2), countdown=10)
tasks.add(2, 2)
```
该任务的签名为**Arity**2(两个参数)：(2，2)，并将倒数执行选项设置为10。
或者您可以使用任务的签名方法创建一个：
```
>>> add.signature((2, 2), countdown=10)
tasks.add(2, 2)
```
还有一个使用**star argument**既s()方法的快捷方式(就是缩写)
```
>>> add.s(2, 2)
tasks.add(2, 2)
```
还支持关键字参数：
```
>>> add.s(2, 2, debug=True)
tasks.add(2, 2, debug=True)
```
在任何签名实例中，您都可以检查不同的字段：
```
>>> s = add.signature((2, 2), {'debug': True}, countdown=10)
>>> s.args
(2, 2)
>>> s.kwargs
{'debug': True}
>>> s.options
{'countdown': 10}
```
它支持延迟，`apply_async`等的“调用API”，包括直接调用(`__call__`)。
```
>>> add(2, 2)
4
>>> add.s(2, 2)()
4
```
**delay**是我们钟爱的apply_async捷径，采用`star argument`：
```
>>> result = add.delay(2, 2)
>>> result.get()
4
```
`apply_async`采用与[`app.Task.apply_async()`](https://docs.celeryproject.org/en/v4.3.0/reference/celery.app.task.html#celery.app.task.Task.apply_async)方法相同的参数：
```
>>> add.apply_async(args, kwargs, **options)
>>> add.signature(args, kwargs, **options).apply_async()

>>> add.apply_async((2, 2), countdown=1)
>>> add.signature((2, 2), countdown=1).apply_async()
```
您无法使用s()定义选项，但是**chaining**调用可以解决此问题：
ps: 一些出现在deply中的option参数不能直接在s()中使用，但是可以通过`set`加入
```或者，您可以在当前过程中直接调用它：
>>> add.s(2, 2).set(countdown=1)
proj.tasks.add(2, 2)
```

### Partials 偏函数
使用签名，您可以在工作线程中执行任务
```
>>> add.s(2, 2).delay()
>>> add.s(2, 2).apply_async(countdown=1)
```
或者，您可以在当前进程中直接调用它：
```
>>> add.s(2, 2)()
4
```
指定其他args，kwarg或apply_async/delay选项可以创建局部变量：
添加的所有参数将在签名中的args之前添加：
```
>>> partial = add.s(2)          # incomplete signature
>>> partial.delay(4)            # 4 + 2
>>> partial.apply_async((4,))  # same
```
添加的所有关键字参数将与签名中的kwargs合并，新关键字参数优先：
```
>>> s = add.s(2, 2)
>>> s.delay(debug=True)                    # -> add(2, 2, debug=True)
>>> s.apply_async(kwargs={'debug': True})  # same
```
添加的所有选项都将与签名中的选项合并，新选项优先：
```
>>> s = add.signature((2, 2), countdown=10)
>>> s.apply_async(countdown=1)  # countdown is now 1
```
您还可以克隆签名以创建派生：
```
>>> s = add.s(2)
proj.tasks.add(2)

>>> s.clone(args=(4,), kwargs={'debug': True})
proj.tasks.add(4, 2, debug=True)
```
### 不变性(?) Immutability
3.0版中的新功能。
`Partials`与`callbacks`一起使用，`tasks linked`和`chord callbacks `都将与父任务的结果一起应用。有时您想指定一个不带其他参数的回调，在这种情况下，您可以将签名设置为不可变的：
```
>>> add.apply_async((2, 2), link=reset_buffers.signature(immutable=True))
```
带有`Immutability`参数签名的方法可以简写成 `si()`
```
>>> add.apply_async((2, 2), link=reset_buffers.si())
```
签名不变时，只能设置执行选项，因此无法使用部分args/kwargs调用签名。
注意 在本教程中，我有时使用前缀运算符〜来签名。您可能不应该在生产代码中使用它，但这是在Python shell中进行实验时的缩写：
```
>>> ~sig

>>> # is the same as
>>> sig.delay().get()
```

### 回调 Callbacks
3.0版中的新功能。
可以使用apply_async的link参数将回调(Callbacks)添加到任何任务：
```
add.apply_async((2, 2), link=other_task.s())
```
仅当任务成功退出时才应用回调，并且回调将以父任务的返回值作为参数来应用。
正如我前面提到的，添加到签名的任何参数都将放在签名本身指定的参数之前！
如果您有签名：
```
>>> sig = add.s(10)
```
然后sig.delay（result）变成：
```
>>> add.apply_async(args=(result, 10))
```
现在，我们使用部分参数通过回调调用添加任务：
```
>>> add.apply_async((2, 2), link=add.s(8))
```
如预期的那样，这将首先启动一个任务计算2+2，然后启动另一个任务计算4+8。


### 原语 The Primitives 
3.0版中的新功能。
总览
group 
组原语是一个签名，其中包含应并行应用的任务列表。
chain
链原语使我们可以将签名链接在一起，以便一个被另一个调用，实质上形成了回调链。
chord
和弦就像一个组，但带有回调。和弦由标题组和主体组成，其中主体是应在标题中的所有任务完成后执行的任务。
map
映射基元的工作方式类似于内置的映射功能，但是会创建一个临时任务，在该任务中将参数列表应用于该任务。例如，task.map（[1，2]）–导致单个任务被调用，将参数应用到任务函数，因此结果为：
```
res = [task(1), task(2)]
```
### starmap
除了将参数应用为*args之外，其工作原理与map完全相同。例如，add.starmap（[[(2，2)，(4，4)]）：
chunks
块将一长串参数分成几部分，例如以下操作：
```
>>> items = zip(range(1000), range(1000))  # 1000 items
>>> add.chunks(items, 10)
```
将项目列表分成10个大块，从而产生100个任务（每个任务依次处理10个项目）。
这些原语本身也是签名对象，因此它们可以以多种方式组合以组成复杂的工作流程。 
这里有一些例子：
简单的链式调用 simple chain
这是一个简单的链，执行第一个任务，将其返回值传递给链中的下一个任务，依此类推。
```
>>> from celery import chain

>>> # 2 + 2 + 4 + 8
>>> res = chain(add.s(2, 2), add.s(4), add.s(8))()
>>> res.get()
16
```
也可以使用管道来编写：
```
>>> (add.s(2, 2) | add.s(4) | add.s(8))().get()
16
```
不变的签名 Immutable signatures
签名可以是部分签名，因此可以将参数添加到现有参数中，但是您可能并不总是这样，例如，如果您不希望链中上一个任务的结果。
在这种情况下，您可以将签名标记为不可变的，这样就不能更改参数：
```
>>> add.signature((2, 2), immutable=True)
```
还有一个.si() 缩写，这是创建签名的首选方式：
```
>>> add.si(2, 2)
```
现在，您可以创建一系列独立的任务：
```
>>> res = (add.si(2, 2) | add.si(4, 4) | add.si(8, 8))()
>>> res.get()
16
>>> res.parent.get()
8
>>> res.parent.parent.get()
4
```
简单组 Simple group
可以轻松创建一组任务以并行执行：
```
>>> from celery import group
>>> res = group(add.s(i, i) for i in range(10))()
>>> res.get(timeout=1)
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```
简单和弦 Simple chord
和弦原语使我们能够添加当组中的所有任务完成执行时要调用的回调。对于不令人尴尬地并行的算法，通常需要这样做：
```
>>> res = chord((add.s(i, i) for i in range(10)), xsum.s())()
>>> res.get()
90
```
上面的示例创建了10个并行启动的任务，当所有任务完成时，返回值组合到一个列表中并发送到xsum任务。
和弦的主体也可以是不变的，因此该组的返回值不会传递给回调：
```
>>> chord((import_contact.s(c) for c in contacts),
...       notify_complete.si(import_id)).apply_async()
```
注意上面使用.si；这将创建一个不变的签名，这意味着传递的任何新参数（包括传递给先前任务的返回值）都将被忽略。

组合
链的偏函数：
```
>>> c1 = (add.s(4) | mul.s(8))

# (16 + 4) * 8
>>> res = c1(16)
>>> res.get()
160
```
这意味着您可以组合链：
```
# ((4 + 16) * 2 + 4) * 8
>>> c2 = (add.s(4, 16) | mul.s(2) | (add.s(4) | mul.s(8)))

>>> res = c2()
>>> res.get()
352
```
将组与另一个任务链接在一起将自动将其升级为和弦：
```
>>> new_user_workflow = (create_user.s() | group(
...                      import_contacts.s(),
...                      send_welcome_email.s()))
... new_user_workflow.delay(username='artv',
...                         first='Art',
...                         last='Vandelay',
...    
```
如果您不想将参数转发给该组，则可以使该组中的签名不变：
```
>>> res = (add.s(4, 4) | group(add.si(i, i) for i in range(10)))()
>>> res.get()
<GroupResult: de44df8c-821d-4c84-9a6a-44769c738f98 [
    bc01831b-9486-4e51-b046-480d7c9b78de,
    2650a1b8-32bf-4771-a645-b0a35dcc791b,
    dcbee2a5-e92d-4b03-b6eb-7aec60fd30cf,
    59f92e0a-23ea-41ce-9fad-8645a0e7759c,
    26e1e707-eccf-4bf4-bbd8-1e1729c3cce3,
    2d10a5f4-37f0-41b2-96ac-a973b1df024d,
    e13d3bdb-7ae3-4101-81a4-6f17ee21df2d,
    104b2be0-7b75-44eb-ac8e-f9220bdfa140,
    c5c551a5-0386-4973-aa37-b65cbeb2624b,
    83f72d71-4b71-428e-b604-6f16599a9f37]>

>>> res.parent.get()
```

### Chains
3.0版中的新功能
可以将任务链接在一起：当任务成功返回时，将调用链接的任务：
```
>>> res = add.apply_async((2, 2), link=mul.s(16))
>>> res.get()
4
```
链接的任务将以其父任务的结果作为第一个参数应用。在上述结果为4的情况下，这将导致mul（4，16）。
结果将跟踪原始任务调用的所有子任务，并且可以从结果实例进行访问：
```
>>> res.children
[<AsyncResult: 8c350acf-519d-4553-8a53-4ad3a5c5aeb4>]

>>> res.children[0].get()
64
```
结果实例还具有[collect()](https://docs.celeryproject.org/en/latest/reference/celery.result.html#celery.result.AsyncResult.collect)方法，该方法将结果视为图形，从而使您可以迭代结果：
```
>>> list(res.collect())
[(<AsyncResult: 7b720856-dc5f-4415-9134-5c89def5664e>, 4),
 (<AsyncResult: 8c350acf-519d-4553-8a53-4ad3a5c5aeb4>, 64)]
```
默认情况下，如果图形未完全形成（其中一项任务尚未完成），[collect()](https://docs.celeryproject.org/en/latest/reference/celery.result.html#celery.result.AsyncResult.collect)将引发[IncompleteStream](https://docs.celeryproject.org/en/latest/reference/celery.exceptions.html#celery.exceptions.IncompleteStream)异常，但是您也可以获取图形的中间表示形式：
```
>>> for result, value in res.collect(intermediate=True):
....
```
您可以将任意多个任务链接在一起，并且签名也可以链接：
```
>>> s = add.s(2, 2)
>>> s.link(mul.s(4))
>>> s.link(log_result.s())
```
您还可以使用`on_error`方法添加错误回调 error callback：
```
>>> add.s(2, 2).on_error(log_error.s()).delay()
```
应用签名时，这将导致以下`.apply_async`调用：
```
>>> add.apply_async((2, 2), link_error=log_error.s())
```
工作者实际上不会将errback作为任务调用，而是直接调用errback函数，以便可以将原始请求，异常和回溯对象传递给它。
这是一个错误示例(误  异步log写入
```
from __future__ import print_function

import os

from proj.celery import app

@app.task
def log_error(request, exc, traceback):
    with open(os.path.join('/var/errors', request.id), 'a') as fh:
        print('--\n\n{0} {1} {2}'.format(
            task_id, exc, traceback), file=fh)
```
为了使将任务链接在一起更加容易，有一个特殊的签名称为chain，可以将任务链接在一起：
```
>>> from celery import chain
>>> from proj.tasks import add, mul

>>> # (4 + 4) * 8 * 10
>>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))
proj.tasks.add(4, 4) | proj.tasks.mul(8) | proj.tasks.mul(10)
```
调用链将调用当前进程中的任务，并返回链中最后一个任务的结果：
```
>>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))()
>>> res.get()
640
```
它还设置了父级属性，以便您可以沿链条向上移动以获得中间结果：
```
>>> res.parent.get()
64

>>> res.parent.parent.get()
8

>>> res.parent.parent
<AsyncResult: eeaad925-6778-4ad1-88c8-b2a63d017933>
```
链也可以使用｜简写 作为一种运算符：
>>> (add.s(2, 2) | mul.s(8) | mul.s(10)).apply_async()

### Graphs
另外，您可以将结果图作为[DependencyGraph](https://docs.celeryproject.org/en/latest/internals/reference/celery.utils.graph.html#celery.utils.graph.DependencyGraph)使用：
```
>>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))()

>>> res.parent.parent.graph
285fa253-fcf8-42ef-8b95-0078897e83e6(1)
    463afec2-5ed4-4036-b22d-ba067ec64f52(0)
872c3995-6fa0-46ca-98c2-5a19155afcf0(2)
    285fa253-fcf8-42ef-8b95-0078897e83e6(1)
        463afec2-5ed4-4036-b22d-ba067ec64f52(0)
```
您甚至可以将这些图形转换为点格式：
```
>>> with open('graph.dot', 'w') as fh:
...     res.parent.parent.graph.to_dot(fh)
```
并创建图像：
```
$ dot -Tpng graph.dot -o graph.png
```
![graph.png](https://docs.celeryproject.org/en/v4.3.0/_images/result_graph.png)

### 组 Groups
3.0版中的新功能。
一个组可用于并行执行多个任务。
[组](https://docs.celeryproject.org/en/v4.3.0/reference/celery.html#celery.group)功能携带签名列表：
```
>>> from celery import group
>>> from proj.tasks import add

>>> group(add.s(2, 2), add.s(4, 4))
(proj.tasks.add(2, 2), proj.tasks.add(4, 4))
```
如果调用该**group**，则将在当前进程中依次应用任务，并返回一个`GroupResult`实例，该实例可用于跟踪结果或告知已准备好多少个任务，依此类推：
```
>>> g = group(add.s(2, 2), add.s(4, 4))
>>> res = g()
>>> res.get()
[4, 8]
```
group还支持迭代器：
```
>>> group(add.s(i, i) for i in xrange(100))()
```
group是签名对象，因此可以与其他签名结合使用。
### roup Results
group 任务也返回一个特殊的**result**，该结果与普通任务结果一样工作，只是它对整个组都有效：
```
>>> from celery import group
>>> from tasks import add

>>> job = group([
...             add.s(2, 2),
...             add.s(4, 4),
...             add.s(8, 8),
...             add.s(16, 16),
...             add.s(32, 32),
... ])

>>> result = job.apply_async()

>>> result.ready()  # have all subtasks completed?
True
>>> result.successful() # were all subtasks successful?
True
>>> result.get()
[4, 8, 16, 32, 64]
```
[GroupResult](https://docs.celeryproject.org/en/v4.3.0/reference/celery.result.html#celery.result.GroupResult)获取[AsyncResult](https://docs.celeryproject.org/en/v4.3.0/reference/celery.result.html#celery.result.AsyncResult)实例的列表，并对其进行操作，就好像这是一个单独的任务一样。
它支持以下操作：
`successful()`
如果所有子任务成功完成（例如，未引发异常），则返回True。
`failed()`
如果任何子任务失败，则返回True。
`waiting()`
如果尚未完成任何子任务，则返回True。
`ready()`
如果所有子任务都准备就绪，则返回True。
`completed_count()`
返回完成的子任务数。
`revoke()`
撤消所有子任务。
`join()` 
收集所有子任务的结果，并以与调用它们相同的顺序返回它们（作为列表）。
ps: （为什么都用join表示顺序施行？）

### Chords
2.3版的新功能。
注意 和弦中使用的任务一定不能忽略其结果。如果对和弦中的任何任务（标题或正文）禁用了结果后端，则应阅读“重要说明”。 RPC结果后端当前不支持和弦。

和弦是仅在组中的所有任务完成执行后才执行的任务。

让我们计算一下表达式的总和（最多一百位数）。

首先，您需要执行两项任务，即add() 和tsum() _(sum()已经是标准函数)_：
```
@app.task
def add(x, y):
    return x + y

@app.task
def tsum(numbers):
    return sum(numbers)
```
现在，您可以使用和弦来并行计算每个加法步骤，然后获取结果数字的总和：
```
>>> from celery import chord
>>> from tasks import add, tsum

>>> chord(add.s(i, i)
...       for i in xrange(100))(tsum.s()).get()
9900
```
显然，这是一个非常人为的示例，消息传递和同步的开销使其比Python慢得多：
```
>>> sum(i + i for i in xrange(100))
```
同步步骤成本很高，因此您应尽可能避免使用和弦。尽管如此，和弦仍然是工具箱中强大的原语，因为同步是许多并行算法所必需的步骤。
让我们分解一下和弦表达式
```
>>> callback = tsum.s()
>>> header = [add.s(i, i) for i in range(100)]
>>> result = chord(header)(callback)
>>> result.get()
9900
```
请记住，只有在头文件中的所有任务都返回之后才能执行回调。header中的每个步骤都作为任务并行执行，可能在不同的节点上执行。然后，在header中将回调与每个任务的返回值一起应用。 **chord()**返回的任务ID是回调的ID，因此您可以等待其完成并获取最终的返回值[（**!!!但请记住，切勿让任何任务等待其他任务**）](https://docs.celeryproject.org/en/v4.3.0/userguide/tasks.html#task-synchronous-subtasks)

### 异常处理 Error handling
那么，如果其中一项任务引发异常怎么办？
和弦回调结果将转换为失败状态，并且错误设置为[ChordError](https://docs.celeryproject.org/en/v4.3.0/reference/celery.exceptions.html#celery.exceptions.ChordError)异常：
```
>>> c = chord([add.s(4, 4), raising_task.s(), add.s(8, 8)])
>>> result = c()
>>> result.get()
```
ps：这是在任务里直接加入了一个必然raise error的任务
```
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "*/celery/result.py", line 120, in get
    interval=interval)
  File "*/celery/backends/amqp.py", line 150, in wait_for
    raise meta['result']
celery.exceptions.ChordError: Dependency 97de6f3f-ea67-4517-a21c-d867c61fcb47
    raised ValueError('something something',)
```
虽然回溯可能会有所不同，具体取决于所使用的结果后端，但是您可以看到错误描述包括失败的任务的ID和原始异常的字符串表示形式。您还可以在`result.traceback`中找到原始的追溯。

请注意，其余任务仍将执行，因此即使中间任务失败，第三个任务（add.s（8，8））仍将执行。此外，[ChordError](https://docs.celeryproject.org/en/v4.3.0/reference/celery.exceptions.html#celery.exceptions.ChordError)仅显示首先（及时）失败的任务：它不遵守标题组的顺序。

要在和弦失败时执行操作，因此可以将errback附加到和弦回调中：
```
@app.task
def on_chord_error(request, exc, traceback):
    print('Task {0!r} raised error: {1!r}'.format(request.id, exc))
```
```
>>> c = (group(add.s(i, i) for i in range(10)) |
...      xsum.s().on_error(on_chord_error.s())).delay()
```
### 重要笔记
和弦中使用的任务一定不能忽略其结果。实际上，这意味着您必须启用`result_backend`才能使用和弦。此外，如果在您的配置中将`task_ignore_result`设置为`True`，请确保要在和弦内使用的各个任务都定义为`ignore_result = False`。这适用于Task子类和修饰的任务。

示例任务子类：
```
class MyTask(Task):
    ignore_result = False
```
装饰任务示例：
```
@app.task(ignore_result=False)
def another_task(project):
    do_something()
```
默认情况下，通过让一个周期性任务每秒轮询一次组的完成，并在准备就绪时调用签名来实现同步步骤。
示例实现： 轮询顺序同步执行就不需要锁
```
from celery import maybe_signature

@app.task(bind=True)
def unlock_chord(self, group, callback, interval=1, max_retries=None):
    if group.ready():
        return maybe_signature(callback).delay(group.join())
    raise self.retry(countdown=interval, max_retries=max_retries)
```
除Redis和Memcached之外，所有结果后端都使用此方法：它们在标头中的每个任务之后增加一个计数器，然后在计数器超过集合中的任务数时应用回调。
Redis和Memcached方法是更好的解决方案，但是在其他后端中不容易实现（欢迎提出建议！）。
注意 在2.2版之前，**chord**无法与Redis一起正常使用；您需要至少升级到redis-server 2.2才能使用它们。

注意 如果您在Redis结果后端使用和弦，并且覆盖了`Task.after_return()`方法，则需要确保调用super方法，否则将不应用和弦回调。
ps： 如果你经常做类似继承的事情 确保调用了父类的super()
```
def after_return(self, *args, **kwargs):
    do_something()
    super(MyTask, self).after_return(*args, **kwargs)
```

### Map & Starmap
map和starmap是内置任务，它们为序列中的每个元素调用该任务。
他们与**group**的不同之处在于
- 仅发送一条任务消息
- 该操作是顺序的

例如使用map
```
>>> from proj.tasks import add

>>> ~xsum.map([range(10), range(100)])
[45, 4950]
```
与执行任务相同：
```
@app.task
def temp():
    return [xsum(range(10)), xsum(range(100))]
```
使用 starmap 实际就是*args map
```
>>> ~add.starmap(zip(range(10), range(10)))
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```
与执行任务相同：
```
@app.task
def temp():
    return [add(i, i) for i in range(10)]
```
map和starmap都是签名对象，因此它们可以用作其他签名，并且可以组合在一起等，例如，在10秒后调用starmap：
```
>>> add.starmap(zip(range(10), range(10))).apply_async(countdown=10)
```

### Chunks
分块可将可迭代的工作分成多个部分，因此，如果您有100万个对象，则可以创建10个任务，每个任务有10万个对象。
有些人可能会担心将任务分块会导致并行度降低，但是对于繁忙的集群而言，情况很少如此，实际上，由于避免了消息传递的开销，因此可能会大大提高性能。
要创建块签名，可以使用[`app.Task.chunks()`](https://docs.celeryproject.org/en/v4.3.0/reference/celery.app.task.html#celery.app.task.Task.chunks)：

```
>>> add.chunks(zip(range(100), range(100)), 10)
```
与**group**一样，发送块消息的行为将在当前进程中在调用时发生：
```
>>> from proj.tasks import add

>>> res = add.chunks(zip(range(100), range(100)), 10)()
>>> res.get()
[[0, 2, 4, 6, 8, 10, 12, 14, 16, 18],
 [20, 22, 24, 26, 28, 30, 32, 34, 36, 38],
 [40, 42, 44, 46, 48, 50, 52, 54, 56, 58],
 [60, 62, 64, 66, 68, 70, 72, 74, 76, 78],
 [80, 82, 84, 86, 88, 90, 92, 94, 96, 98],
 [100, 102, 104, 106, 108, 110, 112, 114, 116, 118],
 [120, 122, 124, 126, 128, 130, 132, 134, 136, 138],
 [140, 142, 144, 146, 148, 150, 152, 154, 156, 158],
 [160, 162, 164, 166, 168, 170, 172, 174, 176, 178],
 [180, 182, 184, 186, 188, 190, 192, 194, 196, 198]]
```
在调用`.apply_async`时，将创建一个专用任务，以便将各个任务应用在工作程序中：
```
>>> add.chunks(zip(range(100), range(100)), 10).apply_async()
```
您还可以将块转换为组：
```
>>> group = add.chunks(zip(range(100), range(100)), 10).group()
```
并且随着组的`skew`，每个任务的倒数以1为增量：
```
>>> group.skew(start=1, stop=10)()
```
这意味着第一个任务的倒计时为一秒，第二个任务的倒计时为两秒，依此类推。