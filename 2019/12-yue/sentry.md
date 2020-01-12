---
description: 部署sentry服务遇到的问题及解决方法
---

# Sentry

## 配置顺序

install之前最好不修改config.yml里的配置，环境变量在docker-compose.yml或者.env里配置，因为config.yml的配置不会被后两者覆盖，从而导致不能修改某些配置。除非替换掉容器里的config.yml。

## 通知

### Email配置

> 163、126邮箱不支持TLS，只支持SSL，而sentry只支持TLS。
>
> 新浪邮箱、qq邮箱、gmail都支持TLS、SSL。

以新浪邮箱为例，在docker-compose.yml中，添加下列环境变量：

```text
    SENTRY_EMAIL_HOST: 'smtp.sina.com'
    SENTRY_EMAIL_USER: 'xxx@sina.com'
    SENTRY_SERVER_EMAIL: 'xxx@sina.com'
    SENTRY_EMAIL_PASSWORD: 'XXX' # 新浪邮箱的客户端授权码
    SENTRY_EMAIL_PORT: 587
    SENTRY_EMAIL_USE_TLS: 'True'
```

### 钉钉

在 [onpremise ](https://github.com/getsentry/onpremise)的 requirements.txt中添加[sentry-dingding](https://github.com/anshengme/sentry-dingding)，然后再install。

然后到钉钉群里创建一个**自定义机器人**，安全设置可以参考[官方文档设置](https://ding-doc.dingtalk.com/doc#/serverapi2/qf2nxq)，这里简单的设置一个alert即可，因为消息的标题格式为\`New alert from XXX'。

![&#x5B89;&#x5168;&#x8BBE;&#x7F6E;](https://github.com/wsfy15/gitbook/tree/0143a2165c437f65a3c445461becd1480ebc2eab/.gitbook/assets/image-1.png)

![&#x6D88;&#x606F;&#x793A;&#x4F8B;](https://github.com/wsfy15/gitbook/tree/0143a2165c437f65a3c445461becd1480ebc2eab/.gitbook/assets/image-2.png)

之后记录下webhook URL的access\_token。

先到具体项目的设置里启用钉钉，然后点击DIngDing的Configure plugin，填入access\_token。然后可以点击右上角测试按钮测试是否成功收到消息。如果没收到，可以先参考[官方文档](https://ding-doc.dingtalk.com/doc#/serverapi2/qf2nxq)的测试自定义机器人部分，排查问题。

![](https://github.com/wsfy15/gitbook/tree/0143a2165c437f65a3c445461becd1480ebc2eab/.gitbook/assets/image-3.png)

![](https://github.com/wsfy15/gitbook/tree/0143a2165c437f65a3c445461becd1480ebc2eab/.gitbook/assets/image-4.png)

