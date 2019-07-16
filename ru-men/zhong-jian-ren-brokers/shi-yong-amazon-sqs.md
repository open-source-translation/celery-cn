# 使用Amazon SQS

##  安装

使用 Amazon SQS 作为中间人（Broker）必须安全一些依赖库，使用 celery\[sqs\] 可以直接安装所有依赖库文件：

```bash
$ pip install celery[sqs]
```

## 配置

使用 Amazon SQS 作为作为中间人的URL为：

```text
broker_url = 'sqs://ABCDEFGHIJKLMNOPQRST:ZYXK7NiynGlTogH8Nj+P9nlE73sq3@'
```

URL 格式为：

```text
sqs://aws_access_key_id:aws_secret_access_key@
```

**注意**：URL 最后必须追加 @ 标识符，URL 才能够正常解析，例如：

```text
from kombu.utils.url import quote

aws_access_key = quote("ABCDEFGHIJKLMNOPQRST")
aws_secret_key = quote("ZYXK7NiynGlTogH8Nj+P9nlE73sq3")

broker_url = "sqs://{aws_access_key}:{aws_secret_key}@".format(
    aws_access_key=aws_access_key, aws_secret_key=aws_secret_key,
)
```

也可以使用环境变量进行设置 AWS\_ACCESS\_KEY\_ID 和 AWS\_SECRET\_ACCESS\_KEY ，在这种情况下中间人（Broker）的配置URL为 sqs:// 。

如果您使用的AWS实例中使用IAM roles，可以将 BROKER\_URL 设置为 sqs:// ，然后 kombu  从实例中的元数据中提取身份令牌信息。

## 设置

### 区域

默认区域为 us-east-1 ，可以通过 broker\_transport\_options 进行自定义区域：

```text
 broker_transport_options = {'region': 'eu-west-1'}
```

{% hint style="info" %}
## See one See （看一看）

Amazon Web Services 区域的详细概述：

[http://aws.amazon.com/about-aws/globalinfrastructure/](http://aws.amazon.com/about-aws/globalinfrastructure/)
{% endhint %}

### 可见性超时

可见性超时为将消息重新下发给另外一个程序之前等待确认的任务秒数。请注意查看下面的[注意事项](shi-yong-amazon-sqs.md#zhu-yi-shi-xiang)。

可以通过 broker\_transport\_options 选项进行修改：

```text
broker_transport_options = {'visibility_timeout': 3600}  # 1 小时
```

默认的可见性超时为 30 分钟。

### 轮训间隔时间

轮训间隔时间为在没有消息的情况获取获取消息的间隔时间，默认值为1秒。值的类型可以为 int 和 float。

可以通过 broker\_transport\_options 选项进行更改默认设置：

```text
broker_transport_options = {'polling_interval': 0.3}
```

 频繁的轮训可能会造成职程（Worker）占用大量的CPU，出现资源浪费的情况。如果需要设置的时间为 亚毫秒，应该可以使用其他的中间人（Broker），例如：RabbitMQ和Redis。

### 队列前缀

默认情况下，Celery 不会设置任务的前缀，可以通过设置 broker\_transport\_options 选项进行设置前缀名称：

```text
 broker_transport_options = {'queue_name_prefix': 'celery-'}
```

## 注意事项

* 如果在“可见性超时“内没有完成任务，该任务会重新分配给另外一个职程（Worker）进行执行。

  这可能会出现在预计时间超出可见性超时时间的问题，如果出现该问题，任务将重新循环执行。

  因此您必须要增加可见性超时时间用于用于匹配最长的执行时间。

  **注意：**Celery 会在职程（Worker）关闭的重新分配消息，如果可见性超时时间过长在断电或者强制终止职程（Worker）的情况会“丢失“重新分配的任务。

  定期执行任务不会被可见性超时影响，因为这是俩个不同的概念。 

  截止撰写文档结束时，AWS支持最大的可见性时间为12个小时\(43200秒\)：

  ```text
  broker_transport_options = {'visibility_timeout': 43200}
  ```

* SQS 不支持远程控制职程（Worker）。
* SQS 不支持事件，相关的clelry 事件、监控等都不支持。

## 结果

Amazon Web Services 系列中有很多产品可以存储结果，但目前没有找到合适的产品进行后端结果。

{% hint style="warning" %}
## 警告

最好不要将 amqp 的结果后端与 SQS 一起使用。

它可能会为每一个任务创建一个消息队列，但创建队列不会搜集。可能会浪费您很多钱钱，最好将后端结果贡献给Celery。
{% endhint %}

