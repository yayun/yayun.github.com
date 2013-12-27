---
layout: post
title: "PMS or source code"
description: "install program in your linux"
category: linux
tags: []
---
{% include JB/setup %}

We may face problems when we install a program .So PMS(package manage system) or source?that is a question.We have two methods when we install program :use the package manager or compiling and installing software from source.

**PMS**

Here is a picture about the process of installing program<!--more-->

<img src="/images/Pms.png"/>



There are so many [package manager](http://en.wikipedia.org/wiki/Package_management_system) that we can use,such as [apt-get](http://en.wikipedia.org/wiki/Advanced_Packaging_Tool) on the Debian GNU/linux distribution,[yum](http://en.wikipedia.org/wiki/Yellow_dog_Updater,_Modified).(如果你下载了.rpm 或者 .deb 的文件 就直接分别可以用RPM( yum) 和dpkg(apt-get) 这两个包管理工具来安装。RPM包本身是有许多依赖的 ，所以在选择上更倾向apt-get 它有以个自动依赖机制。yum 和apt-get是包管理工具 它们分别处理的是rpm和deb包.)We also have [application-level_package_managers](http://en.wikipedia.org/wiki/List_of_software_package_management_systems#Application-level_package_managers),such as [pip](https://pypi.python.org/pypi/pip)(python install package).So which way we can use ? System-level or Application-level? Application-level manager are add-on package managers for operating system with limited capabilities and for programming languages where developers need the latest libraries.So if you want to install specific function software such as Djangoyou can install it use pip.But if you want to install vim,you should use apt-get or yum.Install ruby use APT where install rails use rubygems.

These questions are also applicable to the distribution of our package(System-level vs application-level package):

It seems to me that the procedure followed in the case of applications whose users are happy with they're distributed.


<img src="/images/software_distribute.png"/>

Take, for example:

- Bundler: written in Ruby and useful to Ruby developers; distributed via application-level package.
- fwknop: written in Perl (originally, at least) and useful to end-users; distributed via systems-level packages/installers.
- Pandoc: written in Haskell and may be useful to Haskell developers in the course of developing Haskell applications, but also useful to others; distributed both via application-level and via systems-level packages.


**Source code**

You download a file named pkg.tar.gz. Here is the method,you can also download the file using wget:

    $tar -xvzf pkg.tar.gz
    $cd pkg
    $./configure
    $make
    $make install  # here the make file has been written

The biggest benefit of compiling from source is the ability to pick and choose how your program is built. [For more information:](http://www.hostreview.com/blog/technical_support/articles/sourcecode.html) Compiling from source has its advantages, but if problems occur you are on your own. The apt-get package will, on the other hand, almost always work as well as it can.

**So in conclusion:**

    Pre-Configurated Package: This package is intended for most users. It is designed to work well with most packages, and does not require any extra libraries for compiling.
    Compiled from Source: This package is meant for systems with very customized options. Most importantly, either a custom kernel or custom x-servers/system commands. It is for the more experienced user, but is much more likely to fit any setup, as it is compiled to your system's specs.

**PS:**

Sometimes we may use python setup.py to install or something is simliar to this:

    $tar -xvzf pkg.tar.gz
    $cd pkg
    $python setup.py

In fact python setup.py to install is the analog of make install: it’s a limited way to compile and copy files to destination directories. This doesn’t mean that it’s the best way to really install software on your system.But in my point of view: the best method is use pip.

**Reference:**

[Why you should not use Python's easy_install carelessly on Debian](http://workaround.org/easy-install-debian)

[What is the difference between yum, apt-get, rpm, ./configure && make install](http://superuser.com/questions/125933/what-is-the-difference-between-yum-apt-get-rpm-configure-make-install)






























