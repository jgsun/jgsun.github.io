---
layout: post
title:  "cygwin + jekyll 搭建 github 静态 blog(此文过时)"
categories: build
tags: cygwin, windows10, jekyll,  github page， ruby-devel
author: jgsun
---

[TOC]

* content
{:toc}
# 1.概述
本文讲述 在windows10系统使用cygwin+jekyll搭建github静态博客。











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


（3）关于cygwin 

cygwin是一个在windows平台上运行的类UNIX模拟环境，是cygnus solutions公司开发的自由软件（该公司开发的著名工具还有eCos，不过现已被Redhat收购）。它对于学习UNIX/Linux操作环境，或者从UNIX到Windows的应用程序移植，或者进行某些特殊的开发工作，尤其是使用GNU工具集在Windows上进行嵌入式系统开发，非常有用。随着嵌入式系统开发在国内日渐流行，越来越多的开发者对Cygwin产生了兴趣。(from 百度百科)
> cygwin项目主页 [What... is it?](https://www.cygwin.com/)
```
    a large collection of GNU and Open Source tools which provide functionality similar to a Linux distribution on Windows.
    a DLL (cygwin1.dll) which provides substantial POSIX API functionality.
```

# 2. 搭建步骤
## 2.1 安装工具
（1）安装cygwin及其安装工具apt-cyg
注意安装devel package，详见另外一篇文章：[使用cygwin](https://jgsun.github.io/2019/05/12/using-cygwin/)
（2）安装ruby-devel

`jiangusu@CV0019059N0:~$ apt-cyg install ruby-devel`

查看ruby和gem的版本提示安装成功：
```
jiangusu@CV0019059N0:~$ cd sw/vobs/esam/build/cfnt-b/OS/images/
jiangusu@CV0019059N0:~$ ruby -v
ruby 2.3.6p384 (2017-12-14 revision 9808) [x86_64-cygwin]
last_commit=ruby 2.3.3
jiangusu@CV0019059N0:~$ gem -v
2.6.13
```
（3）安装 jekyll和json

`jiangusu@CV0019059N0:~$ gem install jekyll`

安装完成后将 jekyll  从 ~/.gem/ruby/2.3.0/gems/jekyll-3.8.5/exe目录拷贝到/usr/bin目录，并 查看kelyll版本：
```
jiangusu@CV0019059N0:~/.gem/ruby/2.3.0/gems/jekyll-3.8.5/exe$ cp jekyll /usr/bin/
jiangusu@CV0019059N0:~$ jekyll -v
/usr/share/rubygems/rubygems/core_ext/kernel_require.rb:55:in `require': cannot load such file -- json (LoadError)

...
```
看出报错了，提示没有json，于是安装json：
`jiangusu@CV0019059N0:~/.gem/ruby/2.3.0/gems/jekyll-3.8.5/exe$ gem install json `
安装json之后可以查看到版本,安装成功。

```
jiangusu@CV0019059N0:~$ jekyll -v

jekyll 3.8.5
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


## 2.3  安装 jekyll遇到的问题
错误log：

```
em.cpp: In member function â€˜uint64_t EventMachine_t::GetRealTime()â€™:
em.cpp:448:17: error: â€˜CLOCK_MONOTONIC_RAWâ€™ was not declared in this scope
  clock_gettime (CLOCK_MONOTONIC_RAW, &tv);
                 ^~~~~~~~~~~~~~~~~~~
em.cpp:448:17: note: suggested alternative: â€˜CLOCK_MONOTONICâ€™
  clock_gettime (CLOCK_MONOTONIC_RAW, &tv);
                 ^~~~~~~~~~~~~~~~~~~
                 CLOCK_MONOTONIC
In file included from project.h:46:0,
                 from em.cpp:23:
em.cpp: In static member function â€˜static int EventMachine_t::name2address(const char*, int, int, sockaddr*, size_t*)â€™:
em.cpp:1563:26: warning: comparison between signed and unsigned integer expressions [-Wsign-compare]
   assert (ai->ai_addrlen <= *addr_len);
           ~~~~~~~~~~~~~~~^~~~~
make: *** [Makefile:232: em.o] Error 1


make failed, exit code 2


Gem files will remain installed in /home/jiangusu/.gem/ruby/2.3.0/gems/eventmachine-1.2.7 for inspection.
Results logged to /home/jiangusu/.gem/ruby/2.3.0/extensions/x86_64-cygwin/2.3.0/eventmachine-1.2.7/gem_make.out
```
到目录   /home/jiangusu/.gem/ruby/2.3.0/gems/eventmachine-1.2.7打开 em.cpp文件，根据错误log的建议` em.cpp:448:17: note: suggested alternative: â€˜CLOCK_MONOTONICâ€™ `，将448行的CLOCK_MONOTONIC_RAW改为CLOCK_MONOTONIC（下图红圈所示）之后， make成功。
![image](/images/posts/build/blog-time-src.png)



`jiangusu@CV0019059N0:~/.gem/ruby/2.3.0/gems/eventmachine-1.2.7/ext$ make`
于是保存之后重新执行` gem install jekyll`安装，却还是报同样的错误；原来安装的时候会下载源文件覆盖刚才修改的文件。
执行` gem install jekyll`安装之后，在重新加载窗口出来的时候，迅速点击no和保存按钮，在下载和编译之间打一个时间差把文件内容修改了，编译成功了，安装也成功了。

![image](/images/posts/build/blog-time-src-pop.png)

老版本的cygwin安装jekyll的时候没有这个问题，从cygwin 3.0开始支持CLOCK_MONOTONIC_RAW[What's new and what changed in 3.0](https://www.cygwin.com/cygwin-ug-net/ov-new.html),可能与此相关的cygwin开发库没有更新，所以这里出现了编译错误。可以升级相关开发库或者关闭 CLOCK_MONOTONIC_RAW特性，让GetRealTime走`#elif defined(HAVE_CONST_CLOCK_MONOTONIC)`分支，但是前面的解决方案简单粗暴有效，也没时间去调查研究了。
```
jiangusu@CV0019059N0:~$ ps --version
ps (cygwin) 3.0.1
Show process statistics
Copyright (C) 1996 - 2019 Cygwin Authors
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
## 2.4 修改gem安装源

在某些网络情况下可能需要修改gem安装源， [https://gems.ruby-china.com/](https://gems.ruby-china.com/)主页有详细介绍。
```
$ gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
$ gem sources -l
https://gems.ruby-china.com
# 确保只有 gems.ruby-china.com
```

# 3. 参考资料
* [使用 GitHub, Jekyll 打造自己的免费独立博客](https://blog.csdn.net/on_1y/article/details/19259435)
* [Jekyll 搭建静态博客](https://643435675.github.io/2015/02/15/create-my-blog-with-jekyll/)
*  [手把手教你用github pages搭建博客 最新版](http://www.jianshu.com/p/6fdb19aa4558) 
* [魅族内核团队 meizuosc.github.io ](https://github.com/meizuosc/meizuosc.github.io)
* [搭建一个免费的，无限流量的Blog----github Pages和Jekyll入门](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)
* [CSDN-markdown编辑器语法——背景色](https://blog.csdn.net/testcs_dn/article/details/45766819)
* [gaohaoyang.github.io](https://github.com/Gaohaoyang/gaohaoyang.github.io)

