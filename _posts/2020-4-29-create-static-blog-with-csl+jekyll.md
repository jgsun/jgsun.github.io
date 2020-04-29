---
layout: post
title:  "CSL+jekyll搭建github静态blog"
categories: build
tags: CSL, windows10, jekyll,  github page， ruby-devel
author: jgsun
---

[TOC]

* content
{:toc}
# 1.概述
本文讲述 在windows10系统使用CSL+jekyll搭建github静态博客。











（1）关于github page 

Github Pages提供了一个免费的网页，用来介绍托管在Github上的项目。
由于Github Pages提供免费（300M）、稳定的空间，所以很适合用来创建个人博客。虽然可以使用html来编辑博客，但是显然这样做的工作量比较大，并且博客越复杂就越难维护。庆幸的是，可以通过模板引擎快速创建静态博客。鉴于Github Pages官网推荐了Jekyll模板引擎，下面就介绍如何使用Jekyll来创建博客。(from [使用Jekyll搭建免费的Github Pages个人博客](https://www.jianshu.com/p/abf485c20e3e))
> [What is GitHub Pages?](https://pages.github.com/)

> [Using Jekyll as a static site generator with GitHub Pages](https://help.github.com/en/articles/using-jekyll-as-a-static-site-generator-with-github-pages)


（2）关于jekyll和ruby

Jekyll是一个静态站点生成工具，不需要数据库的支持，通过markdown编写静态文件，生成html页面，并且可以先在本地查看效果，满意之后再提交到Github上，最终在博客主页上看到结果。
由于Jekyll是基于Ruby开发的，所以，要想在本地构建一个Jekyll的测试环境，需要具有Ruby的开发和运行环境。
> jekyll主页: [https://jekyllrb.com/:Transform your plain text into static websites and blogs.](https://jekyllrb.com/)

> [https://rubygems.org/](https://rubygems.org/)
> [https://gems.ruby-china.com/](https://gems.ruby-china.com/)


（3）关于CSL
WSL是微软2016年推出的在windows10运行linx系统，全称是windowsWindows Subsystem for Linux；自从WSL横空出世，我就抛弃了CYGWIN，毕竟WSL是windows原生系统。现在很微软已经发布WSL2，性能更加牛逼，很是期待（因为所用电脑windows10版本还低于安装WSL2的版本）
可到到微软网站WSL主页[Windows Subsystem for Linux Documentation](https://docs.microsoft.com/en-us/windows/wsl/about)了解WSL和安装。

```
The Windows Subsystem for Linux lets developers run a GNU/Linux environment -- including most command-line tools, utilities, and applications -- directly on Windows, unmodified, without the overhead of a virtual machine.
```

# 2. 搭建步骤
## 2.1 安装工具
（1）切换WSL到root用户，安装gcc，g++
```
root@N-20L6PF1QDMGJ:/home/jgsun# apt install gcc 
root@N-20L6PF1QDMGJ:/home/jgsun# apt install g++
```
（2）安装ruby-devel
`gsun@N-20L6PF1QDMGJ:~$ sudo apt-get install ruby-dev`

查看ruby和gem的版本提示安装成功：
```
jgsun@N-20L6PF1QDMGJ:~$ ruby -v
ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-linux-gnu]
jgsun@N-20L6PF1QDMGJ:~$ gem -v
2.7.6
```
（3）安装 jekyll和
`root@N-20L6PF1QDMGJ:/home/jgsun# gem install jekyll`
安装完成后查看kelyll版本：
```
root@N-20L6PF1QDMGJ:/home/jgsun# jekyll -v
jekyll 4.0.0
```
（4）安装 bigdecimal和 paginate
```
gem install bigdecimal
gem install jekyll-paginate
```

## 2.2 启动 jekyll server
进入jgsun.github.io b log目录，启动jekyll服务； 服务成功启动后，访问http://localhost:4000 就可以看到默认的站点主页， 在本地查看效果。

```

jiangusu@CV0019059N0:~$ cd /cygdrive/c/work/blog/jgsun.github.io;jekyll server --watch;
Configuration file: /cygdrive/c/work/blog/jgsun.github.io/_config.yml
            Source: /cygdrive/c/work/blog/jgsun.github.io
       Destination: /cygdrive/c/work/blog/jgsun.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
                    done in 11.424 seconds.
  Please add the following to your Gemfile to avoid polling for changes:
    gem 'wdm', '>= 0.1.0' if Gem.win_platform?
 Auto-regeneration: enabled for '/cygdrive/c/work/blog/jgsun.github.io'
    Server address: http://127.0.0.1:4001/
  Server running... press ctrl-c to stop.
```
然后在浏览器打开http://127.0.0.1:4001/就可以预览了。

# 3. 参考资料
* [使用 GitHub, Jekyll 打造自己的免费独立博客](https://blog.csdn.net/on_1y/article/details/19259435)
* [Jekyll 搭建静态博客](https://643435675.github.io/2015/02/15/create-my-blog-with-jekyll/)
*  [手把手教你用github pages搭建博客 最新版](http://www.jianshu.com/p/6fdb19aa4558) 
* [魅族内核团队 meizuosc.github.io ](https://github.com/meizuosc/meizuosc.github.io)
* [搭建一个免费的，无限流量的Blog----github Pages和Jekyll入门](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)
* [CSDN-markdown编辑器语法——背景色](https://blog.csdn.net/testcs_dn/article/details/45766819)
* [gaohaoyang.github.io](https://github.com/Gaohaoyang/gaohaoyang.github.io)

