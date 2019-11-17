# 最佳实践：Tips and Best Practices

## 忽略不想要的结果

如果你并不关心任务的结果，请务必确定设置 `ignore_result` 选项，因为存储结果会浪费时间和资源。

```python
@app.task(ignore_result=True)
def mytask():
    something()
```

可以使用 `task_ignore_result` 配置全局禁用结果。

在调用`apply_async`和`delay`执行任务时, 通过传递`ignore_result`参数, 可以在每次执行的基础上设置开启/禁用任务结果。

```python
@app.task
def mytask(x, y):
    return x + y

# No result will be stored
result = mytask.apply_async(1, 2, ignore_result=True)
print result.get() # -> None

# Result will be stored
result = mytask.apply_async(1, 2, ignore_result=False)
print result.get() # -> 3
```

默认情况下， 当配置了 `backend` ，任务将不会忽略结果\( `ignore_result=False` \)

选项优先顺序如下\(从低到高\):

* 1.全局选项`task_ignore_result`
* 2.任务配置`ignore_result`
* 3.任务执行时选项`ignore_result`

## 更多的优化技巧

您可以在[优化: Optimizing](../you-hua-optimizing.md)中找到其他的优化技巧。

## 避免启动同步子任务

让一个任务等待另一个任务的结果往往是非常低效的，并在工作池耗尽时，可能会导致死锁。

建议使您的设计异步化，例如使用回调函数。

糟糕的使用:

```python
@app.task
def update_page_info(url):
    page = fetch_page.delay(url).get()
    info = parse_page.delay(url, page).get()
    store_page_info.delay(url, info)

@app.task
def fetch_page(url):
    return myhttplib.get(url)

@app.task
def parse_page(url, page):
    return myparser.parse_document(page)

@app.task
def store_page_info(url, info):
    return PageInfo.objects.create(url, info)
```

较好的使用:

```python
def update_page_info(url):
    # fetch_page -> parse_page -> store_page
    chain = fetch_page.s(url) | parse_page.s() | store_page_info.s(url)
    chain()

@app.task()
def fetch_page(url):
    return myhttplib.get(url)

@app.task()
def parse_page(page):
    return myparser.parse_document(page)

@app.task(ignore_result=True)
def store_page_info(info, url):
    PageInfo.objects.create(url=url, info=info)
```

将不同任务的signature进行链接，组成任务链来达成目的。您可以在[Canvas: 设计工作流程](../canvas-she-ji-gong-zuo-liu-cheng-canvas-designing-workflows/)中了解任务链和其他强大的结构。

默认情况下，Celery不允许您在任务中运行同步子任务，但是在极少数或极端情况下您可能需要这么做。警告：强烈不建议在任务中运行同步子任务。

任务中强制运行同步子任务:

```python
@app.task
def update_page_info(url):
    page = fetch_page.delay(url).get(disable_sync_subtasks=False)
    info = parse_page.delay(url, page).get(disable_sync_subtasks=False)
    store_page_info.delay(url, info)

@app.task
def fetch_page(url):
    return myhttplib.get(url)

@app.task
def parse_page(url, page):
    return myparser.parse_document(page)

@app.task
def store_page_info(url, info):
    return PageInfo.objects.create(url, info)
```

