# 定期任务：Periodic Tasks

## 简介

`celery beat` 是一个调度程序；它定期启动任务，然后由集群中的可用节点执行任务。

默认情况下会从配置中的 `beat_schedule` 项中获取条目\(entries\)，但是也可以使用自定义存储，例如将**entries**存储在SQL数据库中。

应确保一次只运行一个调度程序来执行一个调度程序，否则最终将导致重复的任务。使用集中式方法意味着时间表不必同步，并且该服务可以在不使用锁的情况下运行。

## 时区设置

默认情况下，定期任务计划使用UTC时区，但是可以使用时区设置更改使用的时区。 例如配置 Asia/Shanghai:

```text
timezone = 'Asia/Shanghai'
```

必须通过直接使用\(`app.conf.timezone ='Asia/Shanghai'`\)对其进行配置，或者如果已使用`app.config_from_object`对其进行了设置，则可以将该设置添加到您的应用程序模块\(既常用的`celeryconfig.py`\)中。有关配置选项的更多信息，请参见配置。

默认的调度程序\(将调度程序存储在`celerybeat-schedule`文件中\)将自动检测到时区已更改，并重置调度程序本身，但是其他调度程序可能不那么聪明\(例如**Django**数据库调度程序，请参见下文，在这种情况下，您必须手动重置调度计划。

对于django用户 Celery建议并与**Django 1.4**中引入的新`USE_TZ`设置兼容。 对于Django用户，将使用`TIME_ZONE`设置中指定的时区，或者可以使用celery的`timezone`设置单独为Celery指定自定义时区。 更改与时区相关的设置时，数据库调度程序不会重置，因此您必须手动执行以下操作：

```text
$ python manage.py shell
>>> from djcelery.models import PeriodicTask
>>> PeriodicTask.objects.update(last_run_at=None)
```

**Django-Celery**仅支持Celery 4.0及更低版本，对于Celery 4.0及更高版本，请执行以下操作：

```text
$ python manage.py shell
>>> from django_celery_beat.models import PeriodicTask
>>> PeriodicTask.objects.update(last_run_at=None)
```

## 条目 Entries

要定期调用任务，您必须在周期调度列表中添加一个条目\(**entry**\)。

```python
from celery import Celery
from celery.schedules import crontab

app = Celery()

@app.on_after_configure.connect
def setup_periodic_tasks(sender, **kwargs):
    # Calls test('hello') every 10 seconds.
    sender.add_periodic_task(10.0, test.s('hello'), name='add every 10')

    # Calls test('world') every 30 seconds
    sender.add_periodic_task(30.0, test.s('world'), expires=10)

    # Executes every Monday morning at 7:30 a.m.
    sender.add_periodic_task(
        crontab(hour=7, minute=30, day_of_week=1),
        test.s('Happy Mondays!'),
    )

@app.task
def test(arg):
    print(arg)
```

从`on_after_configure`处理程序中进行设置意味着我们在使用`test.s()`时不会在模块级别评估应用程序。请注意，`on_after_configure`是在设置应用程序后发送的，因此在声明应用程序的模块之外的任务\(例如，在`celery.Celery.autodiscover_tasks()`位于的task.py文件中\)必须使用稍后的信号，例如`on_after_finalize`。 `add_periodic_task()`函数会将条目添加到幕后的`beat_schedule`设置中，并且该设置也可以用于手动设置周期性任务： 例如 每30秒运行task.add任务。

```python
app.conf.beat_schedule = {
    'add-every-30-seconds': {
        'task': 'tasks.add',
        'schedule': 30.0,
        'args': (16, 16)
    },
}
app.conf.timezone = 'UTC'
```

注意 如果想知道这些设置应该去哪里，请参阅[**配置**](https://www.celerycn.io/yong-hu-zhi-nan/pei-zhi-he-mo-ren-pei-zhi-configuration-and-defaults)。您可以直接在应用程序上\(`app.conf.xxx`\)设置这些选项，也可以保留单独的模块\(`celeryconfig.py`\)进行配置。 如果您想对args使用单个项目元组，请不要忘记构造函数是逗号，而不是一对括号。

使用[`timedelta`](https://docs.python.org/dev/library/datetime.html#datetime.timedelta)的时间表表示任务将以30秒的间隔发送（第一个任务将在`celery beat` 开始30秒后发送，然后在最后一次运行之后每30秒发送一次）。

还存在类似Crontab的调度方式，请参阅有关[`Crontab schedules`](https://docs.celeryproject.org/en/stable/userguide/periodic-tasks.html#crontab-schedules)的部分。

与`cron`一样，如果第一个任务在下一个任务之前没有完成，则这些任务可能会重叠。如果对此感到担忧，则应该使用锁定策略来确保一次只能运行一个实例（例如，请确保确保一次只能执行一个任务 [one at one time](https://docs.celeryproject.org/en/stable/tutorials/task-cookbook.html#cookbook-task-serial)）。

## 可用字段

task 要执行的任务的名称。

schedule 执行频率。 这可以是秒数，可以是整数，时间增量\([timedelta](https://docs.python.org/dev/library/datetime.html#datetime.timedelta)\)[crontab](https://docs.celeryproject.org/en/stable/reference/celery.schedules.html#celery.schedules.crontab)。还可以通过扩展计划的界面来定义自己的[自定义计划](https://docs.celeryproject.org/en/stable/reference/celery.schedules.html#celery.schedules.schedule)类型。

args 位置参数（[列表](https://docs.python.org/dev/library/stdtypes.html#list)\)或[元组](https://docs.python.org/dev/library/stdtypes.html#tuple)）

kwargs 关键字参数（[字典](https://docs.python.org/dev/library/stdtypes.html#dict)）

options 执行选项（[字典](https://docs.python.org/dev/library/stdtypes.html#dict)） 这可以是`apply_async()`支持的任何参数 [`exchange`](https://github.com/celery/kombu)，`routing_key`，`expires`等等

relative 如果relative是true，则[timedelta](https://docs.python.org/dev/library/datetime.html#datetime.timedelta)计划是"by the clock."计划的。这意味着频率将根据时间增量的时间取整到最接近的秒，分钟，小时或天。

默认情况下，relative为false，频率不四舍五入，将与开始`celery beat`的时间有关。

## Crontab 调度器

如果要对执行任务的时间（例如，一天中的特定时间或一周中的某天）进行更多控制，则可以使用crontab调取类型：

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    # Executes every Monday morning at 7:30 a.m.
    'add-every-monday-morning': {
        'task': 'tasks.add',
        'schedule': crontab(hour=7, minute=30, day_of_week=1),
        'args': (16, 16),
    },
}
```

这些Crontab表达式的语法非常灵活 一些例子

| Example | Meaning |
| :--- | :--- |
| crontab\(\) | Execute every minute. |
| crontab\(minute=0, hour=0\) | Execute daily at midnight. |

以下略

## Solar 调度

如果您有应根据日出，日落，黎明或黄昏执行的任务，则可以使用[Solar](https://docs.celeryproject.org/en/stable/reference/celery.schedules.html#celery.schedules.solar)调度类型：

```python
from celery.schedules import solar

app.conf.beat_schedule = {
    # Executes at sunset in Melbourne
    'add-at-melbourne-sunset': {
        'task': 'tasks.add',
        'schedule': solar('sunset', -37.81753, 144.96715), # 圣保罗教堂日落 cool~
        'args': (16, 16),
    },
}
```

参数很简单：solar（事件，纬度，经度） 确保对纬度和经度使用正确的符号：

| sign | Argument | Meaning |
| :--- | :--- | :--- |
| + | latitude | North |
| + | latitude | South |
| + | longitude | East |
| + | longitude | West |

可能的事件类型是： 略 [链接](https://docs.celeryproject.org/en/stable/userguide/periodic-tasks.html#crontab-schedules)

所有太阳事件都是使用UTC计算的，因此不受时区设置的影响。 在极地地区，太阳不一定每天都会升起或落下。调度程序可以处理这些情况（例如，在太阳不升起的一天不会发生日出事件）。一个例外是`solar_noon`，正式定义为太阳经过天体子午线的那一刻，即使太阳在地平线以下，它也会每天发生。

暮光定义为黎明到日出之间的时间；在日落和黄昏之间。您可以使用上面列表中的适当事件，根据您对暮光的定义（民用，航海或天文学的）以及是否要在暮光的开始或结束时，根据“暮光”安排事件。 。

有关更多文档，请参见[celery.schedules.solar](https://docs.celeryproject.org/en/stable/reference/celery.schedules.html#celery.schedules.solar)。

## 启动执行计划

启动`celery beat`服务

```text
$ celery -A proj beat
```

您还可以通过启用`workers -B`选项将`celert beat`嵌入到worker内，如果您永远不会运行一个以上的worker节点，这很方便，但是它并不常用，因此不建议用于生产环境：

```text
$ celery -A proj worker -B
```

Beat需要将任务的最后运行时间存储在本地数据库文件（默认情况下命名为`celerybeat-schedule`）中，因此需要访问该文件才能在当前目录中进行写操作，或者您可以为此文件指定一个自定义位置：

```text
$ celery -A proj beat -s /home/celery/var/run/celerybeat-schedule
```

要守护`celery beat`，请参见[守护进程](https://www.celerycn.io/yong-hu-zhi-nan/shou-hu-jin-cheng-daemonization)。

## 自定义调度类

可以在命令行上指定自定义调度程序类（[--scheduler](https://docs.celeryproject.org/en/stable/reference/celery.bin.beat.html#cmdoption-celery-beat-scheduler)参数）。

默认调度程序是`celery.beat.PersistentScheduler`，它仅在本地[slelve](https://docs.python.org/dev/library/shelve.html#module-shelve)文件中跟踪上次运行时间。

还有django-celery-beat扩展程序，用于将调度存储在Django数据库中，并提供了一个方便的管理界面来在运行时管理定期任务。

要安装和使用此扩展： 使用pip安装软件包：

```text
$ pip install django-celery-beat
```

将django\_celery\_beat模块添加到Django项目的settings.py中的INSTALLED\_APPS中：

```text
INSTALLED_APPS = (
    ...,
    'django_celery_beat',
)
```

_请注意，模块名称中没有短划线，只有下划线。_ 应用Django数据库迁移，以便创建必要的表：

```text
$ python manage.py migrate
```

使用`django_celery_beat.schedulers：DatabaseScheduler`调度程序启动`celery beat`服务：

```text
$ celery -A proj beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
```

注意：您也可以直接将其添加为[`beat_scheduler`](https://docs.celeryproject.org/en/stable/userguide/configuration.html#std:setting-beat_scheduler)设置。

请使用Django-Admin界面设置一些定期任务。

