---
layout: post
title: "定制code-server docker image"
category: build
tags: docker-iamge docker-build code-server
author: jgsun
---

* content
{:toc}

# Overview

工欲善其事必先利其器！
前几天想更新下code-server的docker image，修改完成 github 内的 Dockerfile，去 docker hub 查看 image 的 build 情况，发现自动 build iamge 的功能收费了！毕竟天下没有永远免费的午餐，按照同事的说法，其也该收费了。利用其线下 build 功能：线下修改 Dockerfile，build，然后 push到 docker hub，本文记录了这一过程。
![image](/images/posts/code-server/code-server-logo.png)































# 第一步 docker pull codercom/code-server
下 pull coder-server 官方image，自己的image要 from 官方image 开始构建。

    jiangusu@tools$ docker pull codercom/code-server

# 第二步 修改Dockerfile
因为是局域网内 build，需要设置 proxy。
具体 Dockerfile 详见笔者github [Dockerfile](https://github.com/jgsun/code-server/blob/master/Dockerfile)

![image](/images/posts/code-server/dockerfile.png)

# 第三步 docker build image

    jiangusu@code-server$ docker build -t jgsun/code-server:latest  .

# 第四步 docker push 到 docker hub

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