# 写在开头
通过此任务，你将会了解到如何在 Leaflow 上部署一个无状态应用，以及如何将你的应用暴露到互联网中供大家使用。
同时，你还会了解到如何将配置映射变为环境变量，如何使用 Leaflow 集群内部的 LLM 网关。

原 GitHub 仓库：https://github.com/ChatGPTNextWeb/NextChat/blob/main/README_CN.md

部署 NextChat 很简单，仓库中有 Docker Compose，我们根据 Docker Compose 内容就可以了解到一些信息。

1. 先创建一个配置映射

前往 https://leaflow.net/configmaps/create 中，新建一个配置映射，名称设置为 nextchat-config，然后添加以下配置项。

`BASE_URL`, 值为 `http://llm.ai-infra.svc.cluster.local`


2. 创建一个密钥

集群内部的大模型网关需要认证，认证 Token 为您的[访问令牌](https://leaflow.net/openid/tokens)，在右上角创建一个令牌后，复制并保存令牌内容。

随后到 https://leaflow.net/secrets 中，创建一个名叫 `nextchat` 的密钥，密钥类型选择通用密钥。


键名称填写：`OPENAI_API_KEY`, 值为你的访问密钥。

NextChat 还有个访问密码一说，也可以在密钥中创建访问密码。

键名称填写：`CODE`, 值为你想要设置的密码，比如 `123456`

3. 创建工作负载

到 https://leaflow.net/applications/create 中创建一个新的无状态应用，因为我们的工作负载是一个网页应用，所以选择无状态。在工作负载的基本配置中写入名称: `nextchat（名称一定要完全符合，否则无法完成任务），随后副本数量设置为` `1`.

之后到`容器 1` 的基本配置中，输入容器名称为 chatgpt-next-web，镜像填写其官方仓库的 `yidadaa/chatgpt-next-web:latest`。
这个应用理论不会消耗太多的资源，我们保持默认即可。

![容器配置](https://ivampiresp.com/wp-content/uploads/2025/08/image.png)

接着到网络配置中，我们从原本的 Docker Compose 中得知，他的端口为 3000，所以我们也添加一个 3000 端口。

![网络配置](https://ivampiresp.com/wp-content/uploads/2025/08/image-1.png)


之后，我们需要引用来自刚刚创建配置映射和密钥中的键来当做环境变量。

![环境变量](https://ivampiresp.com/wp-content/uploads/2025/08/image-2.png)

我们只需要选择对应的配置名称即可，其余的不用填写，他会自动映射里面的全部内容和键名称作为环境变量。

然后创建即可。

4. 创建服务
无状态应用不会自动创建一个无头服务，所以我们需要到 [服务管理](https://leaflow.net/services) 中创建一个服务。
服务名称写 `nextchat，服务类型选择`，服务类型选择：`内部服务（ClusterIP）`。
然后目标工作负载选择刚刚创建的无状态 nextchat，他会自动填充服务的端口。
随后点击创建服务。

此时创建的服务不能被外部访问，我们这里搭建的是一个网站，所以我们需要创建一个网站。

5. 创建网站

我们这里可以选择使用 `子域名管理器` 来快速创建一个网站用于评估 NextChat。

到 [网站管理](https://leaflow.net/ingresses) 中，点击 `子域名管理器`，然后选择 nextchat 服务，点击创建并等待完成。

6. 享用 NextChat

随后网站管理中会出现 NextChat 的外部访问地址，点击即可访问，然后到 NextChat 的左下角设置中，将访问密码设置为上方设置的“123456”或者你输入的密码，然后即可使用。

恭喜你，你已经学会了 Leaflow 平台部署应用的全流程。
