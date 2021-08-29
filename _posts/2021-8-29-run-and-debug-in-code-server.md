---
layout: post
title: "在code-server中搭建 C++ 开发环境"
category: build
tags: code-server GDB C++ Run-and-Debug
author: jgsun
---

* content
{:toc}

# Overview

作为 low level 的嵌入式工程师，最近准备重学 C++，实际工作中接触 C 代码多一些，实际写 C++ 代码的机会不多，更多的时候是读 C++ 代码。虽然现在 Rust、Python、Go 语言等高级语言风生水起，但 C++ 作为经典的系统级编程语言，在性能、速度方面还是独树一帜，而且理解 C++ 编程思想，对学习新语言也是很有帮助的，比如 Rust 就借鉴和很多 C++ 的思想和语法。公司大部分 application 也是使用 C++ 编写， 熟悉 C++， 进而读懂、熟悉 application，对建立端到端的产品 view 大有裨益。学习掌握 C++ 这件事，就和学习掌握 Python一样， 必须做！

首先要搭建 C++ 的编译、调试环境。一直仅仅使用 code-server 编辑代码，没有使用其编译、调试代码的功能。本文记录了在 code-server 中搭建 C++ 开发环境的过程，实现编辑、编译、GDB 调试一条龙！

![image](/images/posts/code-server/gdb-logo.png)















# 在 code-server docker image 中安装c++扩展插件以及c++编译环境
官方的 Code Server  Docker 缺失对于c++、 c、 python 等的支持，只能用于编辑，不能调试；而在 docker 内使用 apt 安装对应的库后，会在重启 docker 后清除，所以需要通过自己建一个镜像直接包含这些内容。
详见博文 [定制code-server docker image]
# 编辑代码并编译、运行
编辑完代码，点击右上角 run code 按钮，output 窗口显示编译并运行的情况。
![image](/images/posts/code-server/run-code.png)



# Run and Debug
设置好断点，并点击左边 Run and Debug 按钮
![image](/images/posts/code-server/run-and-debug-bp.png)

Terminal报错：`“ptrace operation not permitted`
![image](/images/posts/code-server/run-and-debug-err.png)

## solve “ptrace operation not permitted”
参考文章 [How to solve “ptrace operation not permitted” when trying to attach GDB to a process?](https://stackoverflow.com/questions/19215177/how-to-solve-ptrace-operation-not-permitted-when-trying-to-attach-gdb-to-a-pro)， 添加 docker 运行参数 `--cap-add=SYS_PTRACE --security-opt seccomp=unconfined` 
![image](/images/posts/code-server/docker-run-ptrace.png)


## Debug 效果
![image](/images/posts/code-server/run-and-debug.png)



# 参考资料
* [How to solve “ptrace operation not permitted” when trying to attach GDB to a process?](https://stackoverflow.com/questions/19215177/how-to-solve-ptrace-operation-not-permitted-when-trying-to-attach-gdb-to-a-pro)