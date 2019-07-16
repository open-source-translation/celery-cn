# 使用RabbitMQ

## 安装与配置

RabbitMQ 是默认的中间人（Broker），只需要配置连接的URL即可，不需要安装额外的的配置以及初始化配置信息

```text
broker_url = 'amqp://myuser:mypassword@localhost:5672/myvhost'
```

有关 Celery 各种中间人（Broker）的配置列表，请查阅代理设置，并且按照说明设置用户名和密码。

## 安装 RabbitMQ服务

安装 RabbitMQ 服务可以通过 [RabbitMQ官网](https://www.rabbitmq.com/download.html) 进行 [安装RabbitMQ](https://www.rabbitmq.com/install.html) ，Mac OS安装请查阅 [Mac OS安装RabbitMQ](shi-yong-rabbitmq.md#mac-os-an-zhuang-rabbitmq)。

{% hint style="info" %}
## 注意：

如果在安装 RabbitMQ 后，使用 rabbitmqctl 出现 nodedown 错误信息，可以查阅这片文章解决问题：

[http://www.somic.org/2009/02/19/on-rabbitmqctl-and-badrpcnodedown/](http://www.somic.org/2009/02/19/on-rabbitmqctl-and-badrpcnodedown/)
{% endhint %}

### 配置 RabbitMQ

要使用 Celery，需要创一个RabbitMQ账户：

```bash
$ sudo rabbitmqctl add_user myuser mypassword
$ sudo rabbitmqctl add_vhost myvhost
$ sudo rabbitmqctl set_user_tags myuser mytag
$ sudo rabbitmqctl set_permissions -p myvhost myuser ".*" ".*" ".*"
```

修改myuser、mypassword、myvhost为自己配置的配置信息。

关于更多RabbitMQ配置，请查阅 [RabbitMQ手册](https://www.rabbitmq.com/admin-guide.html)。

### Mac OS安装RabbitMQ

使用Mac OS的 [Homebrew](https://github.com/Homebrew/brew) 安装RabbitMQ最为简单。

首先按照 [Homebrew文档](https://docs.brew.sh/) 提示安装 [Homebrew](https://github.com/Homebrew/brew) ：

```bash
$ sudo ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

然后通过 **brew** 进行安装 RabbitMQ ：

```aspnet
$ brew install rabbitmq
```

使用 **brew** 安装 RabbitMQ 之后需要，将以下代码添加到环境变量（bash\_profile或/etc/profile）中，方便针对中间人（Broker）的管理：

```text
PATH=$PATH:/usr/local/sbin
```

#### 配置系统名称

如果您使用的 DHCP 随机分配的主机名称，需要重新永久化配置主机名称。因为 RabbitMQ 是使用主机名与各个节点进行通信的。

可以使用 **scutil** 命令进行永久配置主机名

```bash
$ sudo scutil --set HostName myhost.local
```

然后将主机名添加到 /etc/hosts 中，以便进行解析

```text
127.0.0.1    localhost myhost myhost.local
```

如果您的 rabbitmq-server 已经在运行，您的节点名称现在应该是 rabbit@myhost ，可以通过 rabbitmqctl 进行验证查看：

```bash
$ sudo rabbitmqctl status
Status of node rabbit@myhost ...
[{running_applications,[{rabbit,"RabbitMQ","1.7.1"},
                    {mnesia,"MNESIA  CXC 138 12","4.4.12"},
                    {os_mon,"CPO  CXC 138 46","2.2.4"},
                    {sasl,"SASL  CXC 138 11","2.1.8"},
                    {stdlib,"ERTS  CXC 138 10","1.16.4"},
                    {kernel,"ERTS  CXC 138 10","2.13.4"}]},
{nodes,[rabbit@myhost]},
{running_nodes,[rabbit@myhost]}]
...done.
```

如果 DHCP 给您分配主机名称是以IP地址（如：23.10.112.31.comcast.net），RabbitMQ将尝试用 raabit@23 ：非法的用户名。

#### 启动/停止 RabbitMQ 服务

启动服务：

```bash
$ sudo rabbitmqctl-server
```

也可以通过添加 -detached 参数在后台运行：

```bash
$ sudo rabbitmqctl-server -detached
```

永远不要通过 **kill** 命令来进行停止 RabbitMQ 运行，使用 rabbitmqctl 命令来进行停止 RabbitMQ ：

```bash
$ sudo rabbitmqctl stop
```

当你服务已经在运行的时候，您可以继续查看 [RabbitMQ配置](shi-yong-rabbitmq.md#pei-zhi-rabbitmq)。



