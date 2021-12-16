---
layout: post
title: "树莓派4 ubuntu 搭建 opengrok standalone 环境"
category: build
tags: pi4 ubuntu opengrok ubuntu tomcat JAVA
author: jgsun
---

* content
{:toc}

# Overview
OpenGrok 是一个很牛的代码阅读浏览工具，用过的都说牛，其官网 <https://oracle.github.io/opengrok> 有详细介绍，不再叙述。
本文介绍在树莓派4(安装 ubuntu 20.04 server 系统) 搭建 opengrok standalone 环境的步骤。
![image](/images/posts/opengrok/opengrok_logo.jpg)



















# 安装 Java
sudo apt install openjdk-17-jdk
# 安装 universal-ctags
sudo apt install universal-ctags
# 下载 tomcat
https://tomcat.apache.org/download-10.cgi

    wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.0-M8/bin/apache-tomcat-10.1.0-M8.tar.gz
    mkdir -p ~/opengrok/tomcat
    tar xvf apache-tomcat-10.1.0-M8.tar.gz -C ../opengrok/tomcat/

## 启动并测试 tomcat
使用下面的命令启动 tomcat：

    cd apache-tomcat-10.1.0-M8
    sh startup.sh

 在浏览器中输入 http://10.10.10.1:8080/， 就看到了 TOM 猫。
![image](/images/posts/opengrok/opengrok_tomcat.jpg)


# 创建 opengrok 目录结构
mkdir -p ~/opengrok/{src,data,dist,etc,log}
# 下载 opengrok tar ball 并解压
First, download the latest version from <https://github.com/oracle/opengrok/releases>

    wget https://github.com/oracle/opengrok/releases/download/1.7.25/opengrok-1.7.25.tar.gz

Second Unpack (assumes GNU tar) the release tarball as follows:

     tar -C ~/opengrok/dist --strip-components=1 -xzf opengrok-1.7.25.tar.gz

# Copy the logging configuration:

    cp ~/opengrok/dist/doc/logging.properties ~/opengrok/etc

# Install management Python opengrok-tools
The python package contains wrappers for OpenGrok's indexer and other commands.

    $ cd ~/opengrok/dist/tools
    $ python3 -m venv env
    $ . ./env/bin/activate
    $ pip install opengrok-tools.tar.gz

这样就可以使用 python3 封装预先定义好的命令，而不需要使用 plain java 命令，如 opengrok-indexer， opengrok-deploy 等。
# Setting up the sources / input data

    git clone https://gitee.com/jgsun/buildroot
    git clone https://gitee.com/jgsun/linux

# Indexing
使用下面的 script 调用 opengrok-indexer 创建 index.
可从网站 <https://gitee.com/jgsun/devkit/blob/master/opengrok/index.sh> 下载。
其中 -i 参数用于 ignore 特地目录，因为有些目录也许 8 辈子都不会看，将它们排除在 index 之外。
```
opengrok-indexer \
    -J=-Djava.util.logging.config.file=~/opengrok/etc/logging.properties \
    -a ~/opengrok/dist/lib/opengrok.jar -- \
    -c /usr/bin/ctags \
    -s ~/opengrok/src -d ~/opengrok/data -H -P -S -G \
    -i d:alpha \
    -i d:arc \
    -i d:arm \
    -i d:c6x \
    -i d:csky \
    -i d:h8300 \
    -i d:h8300 \
    -i d:hexagon \
    -i d:m68k \
    -i d:microblaze \
    -i d:mips \
    -i d:nds32 \
    -i d:nios2 \
    -i d:parisc \
    -i d:powerpc \
    -i d:openrisc \
    -i d:s390 \
    -i d:sh \
    -i d:sparc \
    -i d:um \
    -i d:x86 \
    -i d:xtensa \
    -W ~/opengrok/etc/configuration.xml -U http://localhost:8080/source
```

# Deploy the web application
使用下面的 script 调用 opengrok-deploy Deploy .
```
opengrok-deploy -c ~/opengrok/etc/configuration.xml \
    ~/opengrok/dist/lib/source.war ~/opengrok/tomcat/apache-tomcat-10.1.0-M8/webapps
```
# 使用浏览器打开 http://10.10.10.1:8080/source/ 使用
![image](/images/posts/opengrok/opengrok_browser.jpg)


可见 linux/arch/ 目录下面很多内容被 ignored 了:
![image](/images/posts/opengrok/opengrok_browser_arch.jpg)



# 配置 tomcat 开机启动
(1) 执行如下命令

    sudo mkdir -p /etc/init.d/
    sudo cp ~/opengrok/tomcat/apache-tomcat-10.1.0-M8/bin/catalina.sh /etc/init.d/tomcat
    sudo vi /etc/init.d/tomcat

(2) 在 /etc/init.d/tomcat 开头增加：
```
jgsun@pi4:~$ sudo cat /etc/init.d/tomcat
#!/bin/sh
### BEGIN INIT INFO
# Provides:          tomcat
# Required-Start:    $remote_fs $network
# Required-Stop:    $remote_fs $network
# Default-Start:    2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: The tomcat Java Application Server
### END INIT INFO
CATALINA_HOME=/home/jgsun/opengrok/tomcat/apache-tomcat-10.1.0-M8/
JAVA_HOME=/usr/lib/jvm/java-1.17.0-openjdk-arm64
```
3)更新自启动服务

    sudo update-rc.d -f tomcat defaults

4)开启/停止/查看tomcat服务

    sudo service tomcat start
    sudo service tomcat stop
    sudo service tomcat status


# 参考文档
* [How to setup OpenGrok](https://github.com/oracle/opengrok/wiki/How-to-setup-OpenGrok)
* [Per project management and workflow](https://github.com/oracle/opengrok/wiki/Per-project-management-and-workflow)
* [Ubuntu opengrok 安装新建项目](https://www.jianshu.com/p/2d07abe6e2ae)