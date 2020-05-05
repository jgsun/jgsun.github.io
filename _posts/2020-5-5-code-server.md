---
layout: post
title:  "一种为docker vscode/code-server安装插件的简易方法"
categories: build
tags: vscode, code-server, docker, settings.json, extensions
author: jgsun
---


* content
{:toc}
# 1. Overview
Coder Technologies Inc, an Austin TX company 公司开源了一个基于服务器端的 VScode -- code-server，只要服务器端配置好code-server，就可以在任何浏览器上使用VScode 。code-server目前还不支持在线安装插件，不过它提供了.VSIX方式的安装，本文提供了一种安装插件及配置code-server的简单易行的方法，使用github托管vscode的插件和配置文件，启动docker之后创建插件和配置的软链接即可。不用每次启动docker后离线安装VSIX，也不用重新定制带插件和配置的docker image。



















* 关于code-server： 参考 : https://github.com/cdr/code-server
·code-server is VS Code running on a remote server, accessible through the browser.·
* docker运行code-server，参考: https://hub.docker.com/r/codercom/code-server
# 2. 配置方法
## 2.1 clone https://github.com/jgsun/devkit.git
clone服务器目录 /repo2/jiangusu（目录可以自由指定，需要修改vscode_launch.py为clone的目录）
```
jiangusu@devkit$ pwd
/repo2/jiangusu/devkit
jiangusu@devkit$ git remote -v
origin https://github.com/jgsun/devkit.git (fetch)
origin https://github.com/jgsun/devkit.git (push)
```
## 2.2 拷贝/repo2/jiangusu/devkit/vscode/vscode_launch.py到windows系统
也可以使用WSL在clone到windows系统目录，比如：
```
jgsun@N-20L6PF1QDMGJ:/mnt/c/work/tools/devkit$ git remote -v        
origin https://github.com/jgsun/devkit (fetch)
origin https://github.com/jgsun/devkit (push)
```
## 2.3 SecureCRT运行vscode_launch.py
使用SecureCRT通过ssh登录服务器，然后运行pyghon脚本c:/work/tools/devkit/vscode_launch.py，启动coder-server容器并创建插件目录和配置文件到/repo2/jiangusu/devkit/vscode的相应目录。
![image](/images/posts/code-server/crt-vscode.png)


```
jiangusu@vscode$ /repo2/jiangusu/devkit/vscode/vscode start p=10086
cmd: vscode start p=10086
    Source: /repo2/jiangusu/devkit/vscode
    Config: /home/jiangusu/.vscode/default
    Port: 10086
    container: jiangusu_code_server
47cbfb2ff098a69d4cbf71448d992ea1d30bd3a3ca6b4ea50603d388cc5b18b3
jiangusu@vscode$ /repo2/jiangusu/devkit/vscode/vscode set
setting container, please wait...
jiangusu@vscode$ /repo2/jiangusu/devkit/vscode/vscode login
root@47cbfb2ff098:/home/coder# export src=/repo2/jiangusu/devkit/vscode/
export dst=/root//.localroot@47cbfb2ff098:/home/coder# export dst=/root//.local/share/code-server/
root@47cbfb2ff098:/home/coder# mkdir -p $dst/User/
ln -s $src/settings.json $dst/User/settings.json
ln -s $src/keybindings.json $dst/User/keybindings.json
ln -s $src/extensions $dst/extensions
exit
root@47cbfb2ff098:/home/coder# ln -s $src/settings.json $dst/User/settings.json
root@47cbfb2ff098:/home/coder# ln -s $src/keybindings.json $dst/User/keybindings.json
root@47cbfb2ff098:/home/coder# ln -s $src/extensions $dst/extensions
root@47cbfb2ff098:/home/coder# exit
exit
```
登录docker shell，查看
```
jiangusu@vscode$ docker exec -it jiangusu_code_server bash
root@47cbfb2ff098:/home/coder# ls ~/.local/share/code-server/
coder.json extensions heartbeat logs machineid User
root@47cbfb2ff098:/home/coder# ls ~/.local/share/code-server/ -l
total 8
-rw-r--r-- 1 root root 119 Apr 30 23:56 coder.json
lrwxrwxrwx 1 root root 41 Apr 30 23:54 extensions -> /repo2/jiangusu/devkit/vscode//extensions
-rw-r--r-- 1 root root 0 May 1 00:22 heartbeat
drwxr-xr-x 3 root root 29 Apr 30 23:56 logs
-rw-r--r-- 1 root root 36 Apr 30 23:56 machineid
drwxr-xr-x 5 root root 104 May 1 00:06 User
root@47cbfb2ff098:/home/coder# ls ~/.local/share/code-server/User/ -l
total 0
lrwxrwxrwx 1 root root 47 Apr 30 23:54 keybindings.json -> /repo2/jiangusu/devkit/vscode//keybindings.json
lrwxrwxrwx 1 root root 44 Apr 30 23:54 settings.json -> /repo2/jiangusu/devkit/vscode//settings.json
drwxr-xr-x 2 root root 6 May 1 00:06 snippets
drwxr-xr-x 2 root root 46 Apr 30 23:56 state
drwxr-xr-x 3 root root 22 Apr 30 23:56 workspaceStorage
```
## 2.4 windows系统浏览器登录vscode，打开代码文件夹
可以看见安装的插件和配置都已经生效了。
![image](/images/posts/code-server/extensions.png)


# 3. 升级插件和更新配置
## 3.1 升级插件
使用Visual Studio Code Remote - WSL extension让window版本vscode登录WSL，在线安装插件，然后将安装好的插件commit并push到github，可以完成插件的升级。
![image](/images/posts/code-server/wsl-vscode.png)


```
jgsun@N-20L6PF1QDMGJ:~$ git clone https://github.com/jgsun/devkit
jgsun@N-20L6PF1QDMGJ:~$ cp -rf .vscode-server/extensions/ms-vscode.cpptools-0.27.1 devkit/vscode/extensions/
jgsun@N-20L6PF1QDMGJ:~$ cd devkit/vscode/extensions/
jgsun@N-20L6PF1QDMGJ:~/devkit/vscode/extensions$ ls
hnw.vscode-auto-open-markdown-preview-0.0.4 ms-python.python-2020.2.64397 ms-vscode.go-0.13.0 zainchen.json-1.0.4
kdarkhan.mips-0.0.5 ms-vscode.cpptools-0.26.3 plorefice.devicetree-0.1.1
mrcrowl.hg-1.3.0 ms-vscode.cpptools-0.27.1 technosophos.vscode-make-1.0.2
jgsun@N-20L6PF1QDMGJ:~/devkit/vscode/extensions$ rm -rf ms-vscode.cpptools-0.26.3
jgsun@N-20L6PF1QDMGJ:~/devkit/vscode/extensions$ git add ms-vscode.cpptools-0.27.1
jgsun@N-20L6PF1QDMGJ:~/devkit/vscode/extensions$ git commit
jgsun@N-20L6PF1QDMGJ:~/devkit/vscode/extensions$ git push
jgsun@N-20L6PF1QDMGJ:~/devkit/vscode/extensions$ git log
commit cb00defcab52b447f43a2ca66a24081bd319b05e (HEAD -> master, origin/master, origin/HEAD)
Author: jgsun <jianguo_sun@hotmail.com>
Date: Tue May 5 16:00:22 2020 +0800

    vscode: extensions: update ms-vscode.cpptools to 0.27.0
```
## 3.2 更新配置
 在web界面修改settings.json:
![image](/images/posts/code-server/settings.png)

```
jiangusu@vscode$ git diff
diff --git a/vscode/settings.json b/vscode/settings.json
index 4c34416..f5b9592 100644
--- a/vscode/settings.json
+++ b/vscode/settings.json
@@ -13,6 +13,7 @@
  // "editor.detectIndentation": false, //soft tab
     "editor.mouseWheelZoom": true,
     "explorer.decorations.badges": false,
+ asdf,
     "workbench.colorCustomizations": {
     "editor.selectionHighlightBackground": "#DBA901"
     },
```
下一步commit并push到github。


