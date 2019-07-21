# 过滤配置：Censored configuration

如果您希望将配置作为调试信息或类似信息打印出来，那么您也可能希望过滤掉敏感信息，如密码和API密钥。 Celery 提供了集中打印配置信息工具，其中一个为 `humanize()`：

```bash
>>> app.conf.humanize(with_defaults=False, censored=True)
```

该方法将配置信息转换为列表字符串返回，默认情况下，仅包含修改的键值，可以通过 `with_defaults` 参数进行包含默认的配置信息。 可以通过 `table()` 方法将返回结果转换为字典：

```bash
>>> app.conf.table(with_defaults=False, censored=True)
```

注意：Celery 不会删除所有的敏感配置信息，通过正则表达式来进行检索通常命名的信息，如果包含敏感信息的自定义配置，Celery 会标识为机密的名称来下进行命名秘钥。 

如果命名中含有子字符串，将会进行过滤：

 `API`、`TOKEN`、`KEY`、`SECRET`、`PASS`、`SIGNATURE`、`DATABASE`

