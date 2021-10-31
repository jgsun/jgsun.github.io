---
layout: post
title: "在 code-server Docker Container 使用 host user/group 账户"
category: build
tags: code-server Docker Container Host user group
author: jgsun
---

* content
{:toc}

# Overview
“工欲善其事，必先利其器！”

此文为 [定制 code-server docker image](https://jgsun.github.io/2021/08/29/personlize-code-server-docker-image/) 的续集。
![image](/images/posts/code-server/code-server2.png)













先看当前 code-server Docker Container 的启动命令：

    docker run -ti -d --rm                         \
                   -e PASSWORD=$PROJECT_PASSWORD   \
                   -u root                         \
                   -p $PROJECT_PORT:8080           \
                   --name $CONTAINER_NAME          \
                   jgsun/code-server

可见默认情况下， code-server 容器中的进程以 root 用户执行，并且这个 root 用户和宿主机中的 root 是同一个用户，这意味着：

    容器中运行的进程，在合适的机会下，有权限控制宿主机中的一切；
    容器中运行的进程，以 root 用户执行，外界很难追溯到真实的用户；
    容器进程生成的文件，是 root用户所有，普通用户没有权限读取修改。

可知，以 root 权限执行是很不方便且很不安全的。

于我而言，在 code-server 新建一个源文件，其用户是root，这在回到 host 环境之后维护很麻烦，通常都是下在 host 创建好文件，然后回到 code-server web 界面编辑。所以最好 code-server Docker Container 的用户就是我在服务器 host 的账号。







# Replicate the UID/GID from the host user / group accounts in code-server Docker container
上网一搜，发现许多 Docker 用户有类似的需求，而且 Docker 有解决方案了， 请见参考资料。

[解决 Docker 容器中用户访问权限的问题](https://blog.csdn.net/DreamHome_S/article/details/106543701) 文章总结了两种方案：
1. 在容器内创建用户并且切换用户
2. 使用 fixuid 修改容器中非 root 用户的 uid 和 gid
* About fixuid [github.com/boxboat/fixuid](https://github.com/boxboat/fixuid)
>    fixuid is a Go binary that changes a Docker container's user/group and file permissions that were set at build time to the UID/GID that the container was started with at runtime. Primary use case is in development Docker containers when working with host mounted volumes.
    fixuid was born because there is currently no way to remap host volume UIDs/GIDs from the Docker Engine, see moby issue 7198 for more details.
    Check out BoxBoat's blog post for a practical explanation of how fixuid benefits development teams consisting of multiple developers.
    fixuid should only be used in development Docker containers. DO NOT INCLUDE in a production container image


第一种方案显然不太灵活，在创建 Docker  image 的时候就将用户固定下来了。倾向采用第二种方案，而 coder-server 已经支持第二种方案了，我们缺少的是发现和使用，下面介绍是如何发现并为我所用的。

# 添加启动参数 environment variables "DOCKER_USER=$USER"
浏览 github 上面 [code-server README](https://github.com/cdr/code-server)，找到 [Manually installing code-server](https://coder.com/docs/code-server/latest/install#docker)  docker 安装说明，如下所示：

    # This will start a code-server container and expose it at http://127.0.0.1:8080.
    # It will also mount your current directory into the container as `/home/coder/project`
    # and forward your UID/GID so that all file system operations occur as your user outside
    # the container.
    #
    # Your $HOME/.config is mounted at $HOME/.config within the container to ensure you can
    # easily access/modify your code-server config in $HOME/.config/code-server/config.json
    # outside the container.
    mkdir -p ~/.config
    docker run -it --name code-server -p 127.0.0.1:8080:8080 \
      -v "$HOME/.config:/home/coder/.config" \
      -v "$PWD:/home/coder/project" \
      -u "$(id -u):$(id -g)" \
      -e "DOCKER_USER=$USER" \
      codercom/code-server:latest

对了，就是它了，这正是为了满足我们的需求啊：“使用 host user/group 账户启动 code-server Docker Container。”

于是，Docker Container 的启动命令变为：

    docker run -ti -d --rm                         \
                   -e PASSWORD=$PROJECT_PASSWORD   \
                   -u "$(id -u):$(id -g)"          \
                   -e "DOCKER_USER=$USER"          \
                   -p $PROJECT_PORT:8080           \
                   --name $CONTAINER_NAME          \
                   jgsun/code-server


# 调整 user-data-dir 和 extensions-dir
添加启动参数 environment variables `DOCKER_USER=$USER` 之后，因为变了用户，原来 Dockerfile 里面安装在 root 用户目录的 extensions 和 user settings 都失效了，那就很痛苦了，得固定 extensions 和 user settings 的安装目录让 root 和 DOCKER_USER 都能生效。

Docker 提供了 config.yaml 来配置 environment variables。
1. 修改 Dockerfile 指定 extensions 和 user settings 安装目录：`/home/coder/.local/share/code-server/extensions` 和 `/home/coder/.local/share/code-server/User/`
2. config.yaml 设置 user-data-dir 和 extensions-dir 环境变量与 Dockerfile 指定安装目录一致。

    user-data-dir: /home/coder/.local/share/code-server/
    extensions-dir: /home/coder/.local/share/code-server/extensions

3. 拷贝 config.yaml 到 host 目录 `$HOME/.config/code-server`，并在docker 启动了命令添加目录映射，这样 Docker image 启动的时候可以去读取 config.yaml 里面的 user-data-dir 和 extensions-dir 变量获得 extensions 和 user settings 的安装目录。
```
        if [ "$PROJECT_USER" = "root" ]; then
                PROJECT_OPTIONS="-v "$HOME/.config/code-server:/root/.config/code-server" \
                -u root \
                --cap-add=SYS_PTRACE --security-opt seccomp=unconfined"
        else
                PROJECT_OPTIONS="-v "$HOME/.config/code-server:/home/coder/.config/code-server" \
                -u "$(id -u):$(id -g)" \
                -e "DOCKER_USER=$USER""
        fi
```
完整的 Docker image 启动 script：[https://github.com/jgsun/code-server/blob/master/vscode](https://github.com/jgsun/code-server/blob/master/vscode)

# 修改 DOCKER_USER HOME 目录 owner 和 group
新的问题又来了：现在 coder-server 不是以 root 用户运行，而 user settings 的目录 Dockerfile 在 build 阶段 以 root 用户创建的，这样不能临时修改 settings，因为没有权限保存！需要增加新的解决方案：
1. 复制官方 code-server 的 entrypoint.sh [code-server/ci/release-image/entrypoint.sh](https://github.com/cdr/code-server/blob/main/ci/release-image/entrypoint.sh)
2. 添加修改 DOCKER_USER HOME 目录 owner 和 group 的命令. [code-server/entrypoint.sh](https://github.com/jgsun/code-server/blob/master/entrypoint.sh)

    sudo chown `$DOCKER_USER:$DOCKER_USER $HOME/.config`
    sudo chown -R `$DOCKER_USER:$DOCKER_USER $HOME/.local`

3. Dockerfile 里面添加覆盖官方 code-server Docker image 的 entrypoint.sh 的命令
    `## overwrite entrypoint.sh`
    COPY entrypoint.sh /usr/bin/entrypoint.sh

# 完整使用指南：[code-server/README.md](https://github.com/jgsun/code-server/blob/master/README.md)
## build
    git clone https://github.com/jgsun/code-server
    cd code-server
    docker build -t jgsun/code-server:latest  .

## run
    mkdir -p ~/.config/code-server
    cp config.yaml ~/.config/code-server
    ./vscode start [u=root or $USER] [p=10086]
    Default user is $USER, if want to start by root, add parameter u=root.

## login
    ./vscode login

## stop
    ./vscode stop

# 参考资料
* [Dockerfile将主机用户UID和GID复制到映像](https://www.jb51.cc/docker/437019.html)
* [Docker replicate UID/GID in container from host](https://stackoverflow.com/questions/32397496/docker-replicate-uid-gid-in-container-from-host)
* [解决 Docker 容器中用户访问权限的问题](https://blog.csdn.net/DreamHome_S/article/details/106543701)