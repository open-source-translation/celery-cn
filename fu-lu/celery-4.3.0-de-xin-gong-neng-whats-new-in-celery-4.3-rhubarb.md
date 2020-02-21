# Celery 4.3.0的新功能：What’s new in Celery 4.3 \(rhubarb\)

### What’s new in Celery 4.3 \(rhubarb\)

**作者**:  Omer Katz \(omer.drow at gmail.com\)

`Celery` 是一个用于处理大批量的消息的简单，灵活并且可靠的分布式程序框架，并且提供了以 `Python` 编写的用于维护一个分布式系统必要的工具。

它是一个专注与实时处理的任务队列，并且也支持定时任务。

`Celery` 有一个拥有庞大且多样的用户和志愿者的社区, 你可以通过 [IRC](http://docs.celeryproject.org/en/stable/getting-started/resources.html#irc-channel) 加入我们或者加入 [我们的邮件列表](http://docs.celeryproject.org/en/stable/getting-started/resources.html#mailing-list)

要了解更多关于 `Celery` 的信息，你可以去阅读[介绍](http://docs.celeryproject.org/en/stable/getting-started/introduction.html#intro)

虽然此版本向后兼容之前发布的版本，但阅读下方的章节也是很重要的。

当前版本官方支持 CPython 2.7，3.4，3.5，3.6 & 3.7 ，并且也支持 PyPY2 和 PyPy3。

### 序

4.3.0 版本继续改进我们的工作，以为你提供 `Python` 语言下最好的任务执行平台。

此版本的代号为 [Rhubarb](https://www.youtube.com/watch?v=_AWIqXzvX-U) ， 这是我在 Selected Ambient Works Volume II 最喜欢的曲目之一。

此版本重点关注于新的功能点，例如新的结果后端以及经过改进的安全的序列化器，以及主要针对于 Celery Beat，Canvas的 bug 修复，一些对职程挂起的重要修复和一些针对内存严重泄露的修复。

Celery 4.3 是第一个支持 Python 3.7 的版本。

我们希望 4.3 版本是在我们开始编写 Celery 5 版本，下一代的任务执行平台，时最后一个支持 Python 2.7 的版本。

但是如果 Celery 5 由于一些原因而延迟发布，我们可能会发布另一个 4.x 的版本来支持 Python 2.7。

如果要发布另一个 4.x 的版本，它将会放弃对 Python 3.4 的支持，因为 Python 3.4 的寿命截止到 2019 年 5 月。

我们也致力于解决贡献冲突的问题。

感谢我们的一个核心贡献者 **Josue Balandrano Coronel** ，我们更新了[贡献者文档](https://docs.celeryproject.org/en/v4.3.0/contributing.html#contributing) ，如果你有意贡献代码，请在你方便时间尽快查阅该文档。

我也添加了一些问题的模板，我们将继续对其进行改进，以便于您打开的问题可以包含有利于我们解决问题的更多相关的信息。

---- _Omer Katz_

#### 贡献者墙

Alexander Ioannidis &lt;[a.ioannidis.pan@gmail.com](mailto:a.ioannidis.pan%40gmail.com)&gt; Amir Hossein Saeid Mehr &lt;[amir.saiedmehr@gmail.com](mailto:amir.saiedmehr%40gmail.com)&gt; Andrea Rabbaglietti &lt;[rabbagliettiandrea@gmail.com](mailto:rabbagliettiandrea%40gmail.com)&gt; Andrey Skabelin &lt;[andrey.skabelin@gmail.com](mailto:andrey.skabelin%40gmail.com)&gt; Anthony Ruhier &lt;[anthony.ruhier@gmail.com](mailto:anthony.ruhier%40gmail.com)&gt; Antonin Delpeuch &lt;[antonin@delpeuch.eu](mailto:antonin%40delpeuch.eu)&gt; Artem Vasilyev &lt;[artem.v.vasilyev@gmail.com](mailto:artem.v.vasilyev%40gmail.com)&gt; Asif Saifuddin Auvi &lt;[auvipy@gmail.com](mailto:auvipy%40gmail.com)&gt; Asif Saif Uddin \(Auvi\) &lt;[auvipy@gmail.com](mailto:auvipy%40gmail.com)&gt; Asif Saif Uddin &lt;[auvipy@gmail.com](mailto:auvipy%40gmail.com)&gt; aviadatsnyk &lt;[aviad@snyk.io](mailto:aviad%40snyk.io)&gt; Axel Haustant &lt;[noirbizarre@users.noreply.github.com](mailto:noirbizarre%40users.noreply.github.com)&gt; Benjamin Pereto &lt;[github@sandchaschte.ch](mailto:github%40sandchaschte.ch)&gt; Bojan Jovanovic &lt;[bojan.jovanovic.gtech@gmail.com](mailto:bojan.jovanovic.gtech%40gmail.com)&gt; Brett Jackson &lt;[brett@brettjackson.org](mailto:brett%40brettjackson.org)&gt; Brett Randall &lt;[javabrett@gmail.com](mailto:javabrett%40gmail.com)&gt; Brian Schrader &lt;[brian@brianschrader.com](mailto:brian%40brianschrader.com)&gt; Bruno Alla &lt;[browniebroke@users.noreply.github.com](mailto:browniebroke%40users.noreply.github.com)&gt; Buddy &lt;[34044521+CoffeeExpress@users.noreply.github.com](mailto:34044521+CoffeeExpress%40users.noreply.github.com)&gt; Charles Chan &lt;[charleswhchan@users.noreply.github.com](mailto:charleswhchan%40users.noreply.github.com)&gt; Christopher Dignam &lt;[chris@dignam.xyz](mailto:chris%40dignam.xyz)&gt; Ciaran Courtney &lt;[6096029+ciarancourtney@users.noreply.github.com](mailto:6096029+ciarancourtney%40users.noreply.github.com)&gt; Clemens Wolff &lt;[clemens@justamouse.com](mailto:clemens%40justamouse.com)&gt; Colin Watson &lt;[cjwatson@ubuntu.com](mailto:cjwatson%40ubuntu.com)&gt; Daniel Hahler &lt;[github@thequod.de](mailto:github%40thequod.de)&gt; Dash Winterson &lt;[dashdanw@gmail.com](mailto:dashdanw%40gmail.com)&gt; Derek Harland &lt;[donkopotamus@users.noreply.github.com](mailto:donkopotamus%40users.noreply.github.com)&gt; Dilip Vamsi Moturi &lt;[16288600+dilipvamsi@users.noreply.github.com](mailto:16288600+dilipvamsi%40users.noreply.github.com)&gt; Dmytro Litvinov &lt;[litvinov.dmytro.it@gmail.com](mailto:litvinov.dmytro.it%40gmail.com)&gt; Douglas Rohde &lt;[douglas.rohde2@gmail.com](mailto:douglas.rohde2%40gmail.com)&gt; Ed Morley &lt;[501702+edmorley@users.noreply.github.com](mailto:501702+edmorley%40users.noreply.github.com)&gt; Fabian Becker &lt;[halfdan@xnorfz.de](mailto:halfdan%40xnorfz.de)&gt; Federico Bond &lt;[federicobond@gmail.com](mailto:federicobond%40gmail.com)&gt; Fengyuan Chen &lt;[cfy1990@gmail.com](mailto:cfy1990%40gmail.com)&gt; Florian CHARDIN &lt;[othalla.lf@gmail.com](mailto:othalla.lf%40gmail.com)&gt; George Psarakis &lt;[giwrgos.psarakis@gmail.com](mailto:giwrgos.psarakis%40gmail.com)&gt; Guilherme Caminha &lt;[gpkc@cin.ufpe.br](mailto:gpkc%40cin.ufpe.br)&gt; ideascf &lt;[ideascf@163.com](mailto:ideascf%40163.com)&gt; Itay &lt;[itay.bittan@gmail.com](mailto:itay.bittan%40gmail.com)&gt; Jamie Alessio &lt;[jamie@stoic.net](mailto:jamie%40stoic.net)&gt; Jason Held &lt;[jasonsheld@gmail.com](mailto:jasonsheld%40gmail.com)&gt; Jeremy Cohen &lt;[jcohen02@users.noreply.github.com](mailto:jcohen02%40users.noreply.github.com)&gt; John Arnold &lt;[johnar@microsoft.com](mailto:johnar%40microsoft.com)&gt; Jon Banafato &lt;[jonathan.banafato@gmail.com](mailto:jonathan.banafato%40gmail.com)&gt; Jon Dufresne &lt;[jon.dufresne@gmail.com](mailto:jon.dufresne%40gmail.com)&gt; Joshua Engelman &lt;[j.aaron.engelman@gmail.com](mailto:j.aaron.engelman%40gmail.com)&gt; Joshua Schmid &lt;[jschmid@suse.com](mailto:jschmid%40suse.com)&gt; Josue Balandrano Coronel &lt;[xirdneh@gmail.com](mailto:xirdneh%40gmail.com)&gt; K Davis &lt;[anybodys@users.noreply.github.com](mailto:anybodys%40users.noreply.github.com)&gt; kidoz &lt;[ckidoz@gmail.com](mailto:ckidoz%40gmail.com)&gt; Kiyohiro Yamaguchi &lt;[kiyoya@gmail.com](mailto:kiyoya%40gmail.com)&gt; Korijn van Golen &lt;[korijn@gmail.com](mailto:korijn%40gmail.com)&gt; Lars Kruse &lt;[devel@sumpfralle.de](mailto:devel%40sumpfralle.de)&gt; Lars Rinn &lt;[lm.rinn@outlook.com](mailto:lm.rinn%40outlook.com)&gt; Lewis M. Kabui &lt;[lewis.maina@andela.com](mailto:lewis.maina%40andela.com)&gt; madprogrammer &lt;[serg@anufrienko.net](mailto:serg%40anufrienko.net)&gt; Manuel Vázquez Acosta &lt;[mvaled@users.noreply.github.com](mailto:mvaled%40users.noreply.github.com)&gt; Marcus McHale &lt;[marcus.mchale@nuigalway.ie](mailto:marcus.mchale%40nuigalway.ie)&gt; Mariatta &lt;[Mariatta@users.noreply.github.com](mailto:Mariatta%40users.noreply.github.com)&gt; Mario Kostelac &lt;[mario@intercom.io](mailto:mario%40intercom.io)&gt; Matt Wiens &lt;[mwiens91@gmail.com](mailto:mwiens91%40gmail.com)&gt; Maximilien Cuony &lt;[the-glu@users.noreply.github.com](mailto:the-glu%40users.noreply.github.com)&gt; Maximilien de Bayser &lt;[maxdebayser@gmail.com](mailto:maxdebayser%40gmail.com)&gt; Meysam &lt;[MeysamAzad81@yahoo.com](mailto:MeysamAzad81%40yahoo.com)&gt; Milind Shakya &lt;[milin@users.noreply.github.com](mailto:milin%40users.noreply.github.com)&gt; na387 &lt;[na387@users.noreply.github.com](mailto:na387%40users.noreply.github.com)&gt; Nicholas Pilon &lt;[npilon@gmail.com](mailto:npilon%40gmail.com)&gt; Nick Parsons &lt;[nparsons08@gmail.com](mailto:nparsons08%40gmail.com)&gt; Nik Molnar &lt;[nik.molnar@consbio.org](mailto:nik.molnar%40consbio.org)&gt; Noah Hall &lt;[noah.t.hall@gmail.com](mailto:noah.t.hall%40gmail.com)&gt; Noam &lt;[noamkush@users.noreply.github.com](mailto:noamkush%40users.noreply.github.com)&gt; Omer Katz &lt;[omer.drow@gmail.com](mailto:omer.drow%40gmail.com)&gt; Paweł Adamczak &lt;[pawel.ad@gmail.com](mailto:pawel.ad%40gmail.com)&gt; peng weikang &lt;[pengwk2@gmail.com](mailto:pengwk2%40gmail.com)&gt; Prathamesh Salunkhe &lt;[spratham55@gmail.com](mailto:spratham55%40gmail.com)&gt; Przemysław Suliga &lt;[1270737+suligap@users.noreply.github.com](mailto:1270737+suligap%40users.noreply.github.com)&gt; Raf Geens &lt;[rafgeens@gmail.com](mailto:rafgeens%40gmail.com)&gt; \(◕ᴥ◕\) &lt;[ratson@users.noreply.github.com](mailto:ratson%40users.noreply.github.com)&gt; Robert Kopaczewski &lt;[rk@23doors.com](mailto:rk%4023doors.com)&gt; Samuel Huang &lt;[samhuang91@gmail.com](mailto:samhuang91%40gmail.com)&gt; Sebastian Wojciechowski &lt;[42519683+sebwoj@users.noreply.github.com](mailto:42519683+sebwoj%40users.noreply.github.com)&gt; Seunghun Lee &lt;[waydi1@gmail.com](mailto:waydi1%40gmail.com)&gt; Shanavas M &lt;[shanavas.m2@gmail.com](mailto:shanavas.m2%40gmail.com)&gt; Simon Charette &lt;[charettes@users.noreply.github.com](mailto:charettes%40users.noreply.github.com)&gt; Simon Schmidt &lt;[schmidt.simon@gmail.com](mailto:schmidt.simon%40gmail.com)&gt; srafehi &lt;[shadyrafehi@gmail.com](mailto:shadyrafehi%40gmail.com)&gt; Steven Sklar &lt;[sklarsa@gmail.com](mailto:sklarsa%40gmail.com)&gt; Tom Booth &lt;[thomasbo@microsoft.com](mailto:thomasbo%40microsoft.com)&gt; Tom Clancy &lt;[ClancyTJD@users.noreply.github.com](mailto:ClancyTJD%40users.noreply.github.com)&gt; Toni Ruža &lt;[gmr.gaf@gmail.com](mailto:gmr.gaf%40gmail.com)&gt; tothegump &lt;[tothegump@gmail.com](mailto:tothegump%40gmail.com)&gt; Victor Mireyev &lt;[victor@opennodecloud.com](mailto:victor%40opennodecloud.com)&gt; Vikas Prasad &lt;[vikasprasad.prasad@gmail.com](mailto:vikasprasad.prasad%40gmail.com)&gt; walterqian &lt;[walter@color.com](mailto:walter%40color.com)&gt; Willem &lt;[himself@willemthiart.com](mailto:himself%40willemthiart.com)&gt; Xiaodong &lt;[xd\_deng@hotmail.com](mailto:xd_deng%40hotmail.com)&gt; yywing &lt;[386542536@qq.com](mailto:386542536%40qq.com)&gt;

> 注意:  
> 这里是从 `git` 的提交历史中自动生成的，因此遗憾的是并不包含那些为更重要的事情\(如回答邮件列表里的问题\)提供了帮助的人。

### 从 Celery 4.2 升级

请阅读下方的重要提示，因为其中包含了一些重大更改。

### 重要提示

#### 支持的 Python 版本

支持的 Python 版本有：

* CPython 2.7
* CPython 3.4
* CPython 3.5
* CPython 3.6
* CPython 3.7
* PyPy2.7 6. \(pypy2\)
* PyPy3.5 6.0 \(pypy3\)

#### Kombu

从此版本开始，最低要求的版本是 Kombu 4.4

**新的压缩算法**

Kombu 4.3 包含了一些新的可选的压缩方式：

* LZMA \(在 Python 3 下可以从 stdlib 的获取，或者从向后支持的包中获取\)
* Brotli \(在安装了 brotli 或者 brotlipy 包的情况下可用\)
* ZStandard \(在安装了 zstandard 包的情况下可用\)

不幸的是，我们当前的协议对复杂的 canvas 造成了巨大的负载。

在我们在 Celery 5 中迁移到解决该问题的第 3 版本的 Celery 协议之前，请使用一个新的压缩方式来作为解决办法。

#### **Billiard**

从此版本开始，最低要求的版本是 Billiard 3.6

#### **Eventlet 工作池**

我们要求 `eventlet >= 0.24.1`。

如果你想要使用事件工作池，请使用如下命令安装 Celery：

```text
$ pip install -U celery[eventlet]
```

#### MessagePack 序列化器

一段时间以来，我们并不推荐使用 `msgpack-python` 包。现在我们通过使用 `msgpack` 替代解决了这个问题。

如果你现在正在使用 `MessagePack` 序列化器，请卸载之前的包，并通过如下命令重新安装：

```text
$ pip uninstall msgpack-python -y
$ pip install -U celery[msgpack]
```

#### MongoDB 结果存储

现在我们为 `MongoDB` 结果存储支持了 [DNS seedlist connection format](https://docs.mongodb.com/manual/reference/connection-string/#dns-seedlist-connection-format) 。

这要求安装 `dnspython` 包。

如果你想使用 MongoDB 结果后端，请使用如下安装：

```text
$ pip install -U celery[mongodb]
```

#### Redis 中间人

由于 py-redis 的早期版本中存在多个导致 Celery 出现问题的 bug，我们不得不将最低要求的版本提高到 3.2.0。

**Redis 结果存储**

由于 py-redis 的早期版本中存在多个导致 Celery 出现问题的 bug，我们不得不将最低要求的版本提高到 3.2.0。

#### Riak 结果后端

官方的Riak客户端目前还不支持Python 3.7。

如果您使用的是 Riak 结果后端，请尝试从主分支安装客户端，或者避免升级到Python 3.7，直到解决此问题。

现在在 Python 3.7 环境下使用 Riak 作为结果后端时，我们会报出一个警告。

请追踪 [basho/riak-python-client\#534](https://github.com/basho/riak-python-client/issues/534) 来查看更新。

#### 停止对 RabbitMQ 2.x 的支持

从此版本开始，我们正式不再支持 RabbitMQ 2.x。

2.x 最后一个版本是在 2012 年，我们不得不进行调整来正确支持 RabbitMQ 3.x 的高可用性。

如果因为某些原因，您仍在使用 RabbitMQ 2.x，我们建议您尽快进行升级，因为安全补丁也不再应用于 RabbitMQ 2.x。

#### **Django 支持**

从此版本开始，要求 Django 最低的版本是 1.11。

#### 改进的身份认证序列化器

身份认证序列化器收到了完整的检查，它在之前受到了严重破坏。

我们现在使用 `cryptography` 替代了 `pyOpenSSL`  序列化器。

详细信息请参阅文档 [Message Signing](https://docs.celeryproject.org/en/v4.3.0/userguide/security.html#message-signing) 。

### 新的东西

#### 中间人

**Redis 中间人支持 SSL 地址**

Redis 中间人现在已经支持 SSL 连接。

你可以像平时一样指定 [`broker_use_ssl`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-broker_use_ssl)  来使用 `rediss:// URI.`

你也可以在 URL 中传递 SSL 配置参数：

```text
rediss://localhost:3456?ssl_keyfile=keyfile.key&ssl_certfile=certificate.crt&ssl_ca_certs=ca.pem&ssl_cert_reqs=CERT_REQUIRED
```

**可配置的事件交换机名称**

在之前，事件交换机的名称是硬编码。

你可以使用 [`event_exchange`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-event_exchange) 来指定它。默认情况下保持名称不变。

**可配置的 Pidbox 交换机名称**

在之前，Pidbox 交换机的名称是硬编码。

你可以使用 [`control_exchange`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-control_exchange) 来指定它。默认情况下保持名称不变。

#### **结果存储**

**Redis 结果存储支持 SSL 地址**

Redis 结果存储现在已经可以支持 SSL 连接了。

你可以使用 [`redis_backend_use_ssl`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-redis_backend_use_ssl) 配置来使用 `rediss://URI` 。

你也可以在 URL 中传递 SSL 的配置参数:

```text
rediss://localhost:3456?ssl_keyfile=keyfile.key&ssl_certfile=certificate.crt&ssl_ca_certs=ca.pem&ssl_cert_reqs=CERT_REQUIRED
```

**在结果中存储任务的扩展信息**

在参数 [`result_extended`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-result_extended) 为 `True` 时，结果后端将会存储如下的信息：

* 任务名称
* 参数
* 关键字参数
* 职程执行任务的时间
* 重试的次数
* 队列名称或者路由键

此外，**celery.app.task.update\_state\(\)** 现在可以接受关键字参数，该参数允许您将自定义的数据和任务结果一起存储。

**新的结果后端**

当前版本引入了如下的新的结果存储：

* S3 结果存储
* ArangoDB 结果存储
* Azure Block Blob Storage 结果存储
* CosmosDB 结果存储

**S3 结果存储**

Amazon Simple Storage Service \(Amazon S3\) 是亚马逊提供的对象存储服务。

任务结果将使用如下的路径模板进行存储：

```text
<s3_bucket>/<s3_base_path>/<key>
```

参阅 [S3 backend settings](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#conf-s3-result-backend) 来获得更多信息。

**ArangoDB 结果存储**

ArangoDB 是具有搜索功能单机多模型的数据库。后端将以如下文档的格式存储结果：

```text
{
    _key: {key},
    task: {task}
}
```

参阅 [ArangoDB backend settings](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#conf-arangodb-result-backend) 文档来获取更多信息。

**Azure Block Blob Storage 结果后端**

Azure Block Blob Storage 是微软提供的对象存储服务。

后端将使用如下的路径模板存储结果信息：

```text
<azureblockblob_container_name>/<key>
```

参阅文档 [Azure Block Blob backend settings](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#conf-azureblockblob-result-backend) 来获取更多的信息。

**CosmosDB 结果存储**

Azure Cosmos DB是Microsoft专有的全球分布式多模型数据库服务。

后端将以如下的文档格式存储任务结果：

```text
{
    id: {key},
    value: {task}
}
```

参阅文档 [CosmosDB backend settings \(experimental\)](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#conf-cosmosdbsql-result-backend) 来获取更多信息

#### 任务

**Cythonized 任务**

现在支持了 Cythonized 的任务，你可以将使用 `@task` 装饰器的任务通过 Cython 生成 C 代码，一切将如同源来一样正常的工作。

**确认任务失败或者超时**

在配置 [`task_acks_late`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-task_acks_late) 被设置成 `True` 时，任务将在失败或者超时时被确认。这将死信队列和交换机变得很难使用。

Celery 4.3 引入了新的配置项 [`task_acks_on_failure_or_timeout`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-task_acks_on_failure_or_timeout) 来允许你避免确认那些在配置 [`task_acks_late`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-task_acks_late) 设置为 True 时，出现失败或者超时的任务。

配置 [`task_acks_on_failure_or_timeout`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-task_acks_on_failure_or_timeout) 在默认情况下为 `True` 。

**调度器现在支持微妙**

在使用 Celery beat 计划任务时设置的微妙值不再被忽略。

**默认任务优先级**

你可以通过参数 [`task_default_priority`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-task_default_priority) 设置任务的默认优先级。设置的值将会设置为没有指定优先级的任务的优先级。

**任务可以继承父类的优先级**

将参数 设置为 True 可以使 Celery 的任务继承链接它的前一个任务的优先级。

示例：

```text
c = celery.chain(
  add.s(2), # priority=None
  add.s(3).set(priority=5), # priority=5
  add.s(4), # priority=5
  add.s(5).set(priority=3), # priority=3
  add.s(6), # priority=3
)
```

```text
@app.task(bind=True)
def child_task(self):
  pass

@app.task(bind=True)
def parent_task(self):
  child_task.delay()

# child_task will also have priority=5
parent_task.apply_async(args=[], priority=5)
```

#### **Canvas**

**Chords 可以在 Eager 模式下执行**

在 [`task_always_eager`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-task_always_eager) 设置为 True 时，chords 任务也会急切的运行。

**设置 Chord join 的超时时间**

在之前，**celery.result.GroupResult.join\(\)** 有一个固定的超时时间为 3 秒。

配置项 [`result_chord_join_timeout`](https://docs.celeryproject.org/en/v4.3.0/userguide/configuration.html#std:setting-result_chord_join_timeout) 允许你改变超时时间：

默认值仍是 3 秒。

