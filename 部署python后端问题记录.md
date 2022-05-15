---
title: 部署python后端问题记录
date: 2022-05-06 10:42:08
tags:
---

这学期人工智能要用到python，但是对于不同的电脑上各种python的版本不同，以及3.6与3.9又有着很大区别，python版本更新的速度非常之快，并且大部分用户电脑上都是自带的python2.7版本，同时一般用户又会下载新的python3版本，但是又不会修改默认的python解释器，导致下载库，python解释程序时各种版本争相齐放，共同处理，场面实在太过混乱，因此在这里系统的总结一下，用python2.7来开发后端难免会碰到一些比较新的包比如flask，request，pygame版本支持不同，导致使用起来非常滴难受，特此记录一下。



### 1.首先先记录一下碰到的yum的问题

开始的时候yum出现这样一个报错

```python
yum install libatomic1 -y
There was a problem importing one of the Python modules
required to run yum. The error leading to this problem was:

   /usr/lib64/python2.7/site-packages/pycurl.so: undefined symbol: CRYPTO_num_locks

Please install a package which provides this module, or
verify that the module is installed correctly.

It's possible that the above module doesn't match the
current version of Python, which is:
2.7.5 (default, Aug  4 2017, 00:39:18) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-16)]

If you cannot solve this problem yourself, please go to 
the yum faq at:
  http://yum.baseurl.org/wiki/Faq
```

查询之后猜想yum应该是用c语言写的，yum运行时其所需要的动态链接库的链接出了问题，报错页给出了文件名/usr/lib64/python2.7/site-packages/pycurl.so

然后，我们可以用ldd命令查看这个文件 在 ldd 

注意J：命令打印的结果中，“=>”左边的表示该程序需要连接的共享库之 so 名称，右边表示由 Linux 的共享库系统找到的对应的共享库在文件系统中的具体位置。默认情况下， /etc/ld.so.conf 文件中包含有默认的共享库搜索路径。

然后就去查看了我的 /etc/ld.so.conf 文件，发现里多了一条路径

<img src="/Users/zhangkuang/Documents/College/kuangkaungkaungha/blog/source/_posts/部署python后端问题记录/:etc:ld.so.conf .png" alt=":etc:ld.so.conf " style="zoom:33%;" />

第二条应该是opengauss的依赖库，应该是当时安装opengauss的时候自动加上的，在修改之前libcurl.so.4寻找到的动态库路径就是这个，于是吧这条路径删除，就可以正确找到路径了。

![ldd](/Users/zhangkuang/Documents/College/kuangkaungkaungha/blog/source/_posts/部署python后端问题记录/ldd.png)





### 2.链接问题

[ldd命令详解](https://blog.csdn.net/qq_26819733/article/details/50610129)

通常情况下，许多开放源代码的程序或函数库都会默认将自己安装到 /usr/local 目录下的相应位置（如：/usr/local/bin 或 /usr/local/lib），以便与系统自身的程序或函数库相区别。而许多 Linux 系统的 /etc/ld.so.conf 文件中默认又不包含 /usr/local/lib。因此，往往会出现已经安装了共享库，但是却无法找到共享库的情况。具体解决办法如下：





### 3.系统中的python版本

- 先查看系统中有那些Python版本：


```
ls /usr/bin/python*
whereis python
ls /usr/bin/python*
```



- 再查看系统默认的Python版本：


```
which python
python --version
```

​		

然后在网上找了一个还比较全的教程安装python3.9

[python源码安装](https://segmentfault.com/a/1190000041232486)

​			 				 				 				 			

在linux要查找某个文件，但不知道放在哪里了，可以使用下面的一些命令来搜索： 

- `which` - 查看可执行文件的位置。
- `whereis` - 查看文件的位置。 
- `locate` - 配合数据库查看文件位置。
- `find` - 实际搜寻硬盘查询文件名称。

`which`命令的作用是，在`PATH`变量指定的路径中，搜索某个系统命令的位置，并且返回第一个搜索结果。也就是说，使用`which`命令，就可以看到某个系统命令是否存在，以及执行的到底是哪一个位置的命令。





### 4.linux路径

罗列一下linux下 /usr 目录中的内容（Unix System resource）

- /usr/bin
- /usr/include
- /usr/lib
- /usr/share
- /usr/local
- /usr/src

'**/usr/bin**' 目录包含对所有用户都适用的非基础命令，如果你在 **/bin** 目录下未发现的命令，可以在这里找到。它包含大量的命令程序。

**'/bin'**目录包内 用户的二进制文件以或者可执行文件，包含在单用户模式下使用的 Linux 命令，以及所有用户使用的常见命令，例如 cat，cp，cd，ls 等。

**'/sbin'** 目录还包含可执行文件，但和 '/bin' 不同的是，它仅仅包含系统二进制文件，这些文件需要 root 权限才能执行，而且有助于系统正常运行。例如：fsck、root、init、ifconfig 等等

**'/lib'** 目录包含共享库，它们通常被 '/bin' 和 ‘/sbin' 目录使用。它们还包含内核模块，这些文件名称可标识为 ld* 或 lib*.so.*。例如：ld-linux.so.2 和 libfuse.so.2.8.6





### 5.ubuntu同时安装python2和python3版本的gunicorn

------

### 前言

最近在学习使用 gunicorn 部署 flask 项目。发现使用 pip3 安装完 gunicorn后，如如果再使用 pip2 安装 gunicorn，后安装的 gunicorn 就会覆盖掉原来的，现在将我的解决方案记录一下，留作参考使用。

### 解决方案

1. 卸载全部 gunicorn
   `pip2 uninstall gunicorn`
   `pip3 uninstall gunicorn`
2. 安装 python3 版本的 gunicorn
   (1). `pip3 install gunicorn`
   (2). 使用 `whereis gunicorn` 找到 gunicorn 的位置，我的是在 `/usr/local/bin/gunicorn`.
   (3). 然后进入到这个目录，重命名 gunicorn: `mv gunicorn gunicorn3`
   (4). 在终端输入`gunicorn3 -h`，调用成功，即表示更改成功
3. 安装 python2 版本的 gunicorn
   (1). `pip2 install gunicorn`
   (2). 安装完成后，会发现输入 `gunicorn + Tab键`，gunicorn、gunicorn3同时存在
   (3). 在终端输入`gunicorn -h`，调用成功，即表示安装成功

至此，全部安装完毕，使用 gunicorn3 默认调用 python3，使用 gunicorn 默认调用 python2
注：如果重新安装python2 的 pip 工具，可以参考





### 6.修改python默认版本的多种方法

[这篇博客](https://blog.csdn.net/White_Idiot/article/details/78240298)







