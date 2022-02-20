---
layout: post
title: "优化 code-server Dockerfile"
category: build
tags: Dockerfile docker-build extensions settings.json
author: jgsun
---

* content
{:toc}

# Overview
“工欲善其事，必先利其器！” 
话说上次将 code-server 打造成了集 C/C++/Python 编辑，编译和 GDB debug 于一身的利器，最近却有近两个月没得用，还真有些不习惯。

现在服务器不支持启动 Docker，在订购的个人 workstation 到货，安装好统一的 Ubuntu 系统之后，俺就马上安装 Docker， 启动了 code-server，没错，还是熟悉的味道~

保存在 [docker.io/jgsun/coder-server](https://hub.docker.com/repository/docker/jgsun/code-server) 的 image 还可以使用，但是有一段时间没有更新了，而且还想安装几个有用的 extension， 比如 `GitLens — Git supercharged` 等，所以就开动 docker build “机器”，构建新的 docker image。在这过程中，遇到和解决了一些新的问题，借此机会优化了 Dockerfile. [点我查看新版Dockerfile](https://github.com/jgsun/code-server/blob/master/Dockerfile)

![image](/images/posts/code-server/wow.png)








# 优化 Dockerfile
此定制 docker image 的两大功能是：预先安装需要使用的各种 extensions 和 预先定义各种 Settings；而且需要以 forward host USER 的方式启动 docker，以使在 docker 内修改的文件属性和 host 保持一致，参考 [https://coder.com/docs/code-server/latest/install#docker](https://coder.com/docs/code-server/latest/install#docker)。

Docker image build 的时候 extensions 文件和 Settings 文件的用户名和组都是 root，这就使得以 host USER 启动的 docker 内 extensions 和 settings 不生效，而且也不能修改，首先不生效是必须解决关键功能问题；其次不能修改也不便于使用，万一使用中需要调整 Settings， 那就非常不灵活了！

## Docker build 时指定 extensions 文件 和 Settings 文件的用户名和组
此前版本在 entrypoint.sh 中修改 docker image 内文件夹 /home/coder/.config 和 /home/coder/.local 的用户名和组为 host USER 对应的用户名和组，在 docker build 时覆盖原生 entrypoint.sh。

现在重新 build 得到的 image，docker 启动失败，即使不做任何修改，仅仅覆盖也不行，反之 docker build 时不覆盖就可以启动成功。

必须另辟蹊径！既然 docker 启动的时候不能修改，在 docker build 时就修改好也可以啊！

修改如下，这里直接引用了 Dockerfile 里面的注释：

1. We have to install extensions as host UID:GID so the code-server can only identify the extensions when we start the container by forwarding host UID/GID later. 
2. Because of taking user by `$UID:$GID`, container can't identify the HOME(`~`) variable when building, so we need to declare HOME explicitely, or else hit err "info  Wrote default config file to ~/.config/code-server/config.yaml" 
3. The user and group will be root and the setting won't go into effect before changing user:group to `$UID:$GID` or changing the file mode bits to 777. So we change user:group to `$UID:$GID` when do COPY.

需要在 docker build 的时候将 host UID 和 GID 传递给 Dockerfile， 所以新的 build 命令为：
`docker build --build-arg UID=$(id -u) --build-arg GID=$(id -g) -t jgsun/code-server:latest .`

部分代码段如下：
```
## We have to install extensions as host UID:GID so the code-server can only identify the extensions when we start
## the container by forwarding host UID/GID later.
USER $UID:$GID
## Because of taking user by $UID:$GID, container can't identify the HOME(~) variable, so we need to
## declare HOME explicitely, or else hit err "info  Wrote default config file to ~/.config/code-server/config.yaml" 
RUN HOME=/home/coder code-server \
    --user-data-dir=/home/coder/.local/share/code-server \
    --install-extension ms-vscode.cpptools.vsix \
    --install-extension EugenWiens.bitbake.vsix \
    --install-extension plorefice.devicetree.vsix \
    --install-extension tomoki1207.pdf.vsix \
    --install-extension whiteout2.arm64.vsix \
    --install-extension ms-python.python \
    --install-extension formulahendry.code-runner

## The user and group will be root and the setting won't go into effect before changing user:group to $UID:$GID or changing
## the file mode bits to 777.
COPY --chown=$UID:$GID settings.json /home/coder/.local/share/code-server/User/settings.json
COPY --chown=$UID:$GID keybindings.json /home/coder/.local/share/code-server/User/keybindings.json
```



## 新的 vsix 安装方式
有些 extension 比较特别，在 docker build 的时候根据 Identifier 安装会失败，此前是 build 先使用 wget 下载 vsix，然后安装，现在一些 extension 的官方不在 release vsix 安装包，Microsoft 的 extension Marketspace 好像不太支持 wget 下载，所以采用预先下载 vsix，build 的时候拷贝到 container 之后安装的新方式。
```
## The latest version don't release vsix file in the github, so we download vsix from Microsoft
## https://marketplace.visualstudio.com/VSCode and copy them into container when building.
## RUN wget https://github.com/microsoft/vscode-cpptools/releases/download/1.7.1/cpptools-linux.vsix
COPY vsix/* /home/coder/
RUN HOME=/home/coder code-server \
    --user-data-dir=/home/coder/.local/share/code-server \
    --install-extension ms-vscode.cpptools.vsix 

RUN rm -f *.vsix && rm -rf /home/coder/.local/share/code-server/CachedExtensionVSIXs
```

# 修改 Lanuch 脚本 vscode
1. 因为权限问题调整了目录挂载设置；
2. 增加 `--user-data-dir` 参数指定 extensions 和 uset settings, 因此不再需要在 config.yaml 内的设置 user-data-dir 和 extensions-dir。参考 [How can I reuse my VS Code configuration?](https://coder.com/docs/code-server/latest/FAQ#how-can-i-reuse-my-vs-code-configuration)