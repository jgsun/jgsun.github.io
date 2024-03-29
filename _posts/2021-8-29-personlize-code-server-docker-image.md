---
layout: post
title: "定制 code-server docker image"
category: build
tags: docker-iamge docker-build code-server
author: jgsun
---

* content
{:toc}

# Overview

”工欲善其事, 必先利其器！“

前几天想更新下code-server的docker image，修改完成 github 内的 Dockerfile，去 docker hub 查看 image 的 build 情况，发现自动 build iamge 的功能收费了！毕竟天下没有永远免费的午餐，按照同事的说法，其也该收费了。利用其线下 build 功能：线下修改 Dockerfile，build，然后 push到 docker hub，本文记录了这一过程。
![image](/images/posts/code-server/code-server-logo.png)































# 第一步 docker pull codercom/code-server
下 pull coder-server 官方image，自己的image要 from 官方image 开始构建。

    jiangusu@tools$ docker pull codercom/code-server

# 第二步 修改Dockerfile
因为是局域网内 build，需要设置 proxy。
具体 Dockerfile 详见笔者github [Dockerfile](https://github.com/jgsun/code-server/blob/master/Dockerfile)

![image](/images/posts/code-server/dockerfile.png)

## How to get extention Identifier?
在上述 Dockerfile 安装 extension 时候需要指定extention Identifier，据微软docs [Extension Marketplace]() `When identifying an extension, provide the full name of the form publisher.extension, for example ms-python.python.` 在 
举个例子，想要安装一个 ARM assembly highlighting extension，步骤如下：
（1）在微软 [Extension Marketplace](https://marketplace.visualstudio.com/vscode) 搜索 arm
![image](/images/posts/code-server/extension-search.png)


（2）点击打开 ARM extension 的 detail page，在 more info 区域里面找到 Unique Identifier
![image](/images/posts/code-server/extension-id.png)

（3）在 Dockerfile 里面指定安装 ARM extension: `--install-extension dan-c-underwood.arm`
![image](/images/posts/code-server/extension-installed.png)


# 第三步 docker build image
    
    $ git clone https://github.com/jgsun/code-server
    $ docker build -t jgsun/code-server:latest  .
    Sending build context to Docker daemon  56.18MB
    Step 1/11 : FROM codercom/code-server:latest

# 第四步 启动code-server
这里下载 [devkit/vscode/vscode](https://raw.githubusercontent.com/jgsun/devkit/master/vscode/vscode) 启动 code-server 的 script，并启动 docker container，示例如下：

    $ wget https://raw.githubusercontent.com/jgsun/devkit/master/vscode/vscode
    $ chmod 755 vscode
    $ ./vscode start p=10086
    cmd: vscode start p=10086
    Source: /repo/jiangusu/tools/code-server
    Config: /home/jiangusu/.vscode/default
    Port: 10086
    container: jiangusu_code_server
    4b9f944211a2f4a4348cde3d66102bd7c3085e2f482a85e6b8bec6bfa2563f5c

在浏览器中输入 `http://<server ip>:10086` 打开网页版本的 vscode，可见在已安装的 extension 里面有我们刚刚在 Dockerfile 里面安装的 `ARM extension` 了。

# 第五步 docker push 到 docker hub
将本地 build 的docker image push 到 docker hub，可在其它地方或者供他人 pull 使用。

    jiangusu@code-server$ docker login
    Authenticating with existing credentials...
    WARNING! Your password will be stored unencrypted in /home/jiangusu/.docker/config.json.
    Configure a credential helper to remove this warning. See
    https://docs.docker.com/engine/reference/commandline/login/#credentials-store

    Login Succeeded
    jiangusu@code-server$ docker push jgsun/code-server:latest
    The push refers to repository [docker.io/jgsun/code-server]
    548e96ced8c3: Pushed 
    acde2338b7ac: Pushed 
    6a081071e19b: Pushed 

# 参考资料
* [自行创建Docker镜像](https://nekiglacier.top/2020/10/21/%E8%87%AA%E8%A1%8C%E5%88%9B%E5%BB%BADocker%E9%95%9C%E5%83%8F/)