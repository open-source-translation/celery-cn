# 配置：Configuration

您可以设置几个选项来更改 Celery 的工作方式。这些选项可以直接在app实例上设置，也可以使用专用的配置模块。

你可以设置几个选项来改变 Celery 的工作方式。这些选项可以直接在应用程序实例上设置，也可以使用专用的配置模块。

可以通过配置几个选项来进行配置 Celery，该配置通过直接在程序实例中配置，也可以通过专用的配置模块进行配置。 配置如下 `app.conf`：

```bash
>>> app.conf.timezone
'Europe/London'
```

也可以直接设置对应的值:

```bash
>>> app.conf.enable_utc = True
```

也可以通过 `update` 进行一次设置多个键值：

```bash
>>> app.conf.update(
...     enable_utc=True,
...     timezone='Europe/London',
...)
```

配置对象由多个字典组成，按以下顺序进行查询:

1. 在运行时所做的更改。
2. 配置模块\(如果有的话\)
3. 默认配置\(celery.app.defaults\) 也通过 `app.add_defaults()` 进行配置新的默认配置。

{% hint style="info" %}
## See one see （可以看一看）

详情请参阅，[`配置和默认配置：Configuration and defaults`](../pei-zhi-he-mo-ren-pei-zhi-configuration-and-defaults.md)。
{% endhint %}

## config\_from\_object

`config_from_object()` 可以从配置对象中进行加载配置。 

加载的内容可以为配置模块、也可以为用于属性的对象。 

**注意**：`config_from_object()` 在进行调用时会恢复默认的配置，如果需要设置其他的配置，建议在调用完毕之后进行操作。

### 案例1：使用模块名称

`app.config_from_object()` 可以从一个 python 模块中（包含属性名称）进行加载，例如：`celeryconfig` 、`myproj.config.celery` 或 `myproj.config:CeleryConfig`：

```python
from celery import Celery

app = Celery()
app.config_from_object('celeryconfig')
```

其中 `celeryconfig` 模块内容如下： 

{% code title="celeryconfig.py" %}
```python
enable_utc = True
timezone = 'Europe/London'
```
{% endcode %}

导入 celeryconfig 程序就可以应用。

### 案例2：配置模块对象

可以配置模块对象，但不建议这样用。

{% hint style="info" %}
## Tip

建议使用模块名称，这样在使用 prefork 池的时候不需要进行序列化，如果配置错误或者出现 `pickle` 问题，可以尝试使用模块的名称。
{% endhint %}

```python
import celeryconfig

from celery import Celery

app = Celery()
app.config_from_object(celeryconfig)
```

### 案例3：使用配置类/对象

```python
from celery import Celery

app = Celery()

class Config:
    enable_utc = True
    timezone = 'Europe/London'

app.config_from_object(Config)
# or using the fully qualified name of the object:
#   app.config_from_object('module:Config')
```

## config\_from\_envvar

`app.config_from_envvar()` 可以从环境变量获取信息进行配置， 例如，从名称为 `CELERY_CONFIG_MODULE` 的环境变量中加载配置：

```python
import os
from celery import Celery

#: Set default configuration module name
os.environ.setdefault('CELERY_CONFIG_MODULE', 'celeryconfig')

app = Celery()
app.config_from_envvar('CELERY_CONFIG_MODULE')
```

然后通过指定的环境变量进行配置使用的配置模块：

```bash
$ CELERY_CONFIG_MODULE="celeryconfig.prod" celery worker -l info
```

