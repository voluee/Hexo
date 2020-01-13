---
title: 【reprinted】Docker principle
date: 2019-4-5 12:00:00
top: false
cover: true
toc: true
summary: 介绍容器、镜像的区别和Docker每个命令后面的技术细节
categories: DevOps
tags:
  - Docker

---

# 【转载】深入理解Docker原理

>本文用图文并茂的方式介绍了容器、镜像的区别和Docker每个命令后面的技术细节，能够很好的帮助读者深入理解Docker。

## Image Definition

镜像（Image）就是一堆只读层（read-only layer）的统一视角，也许这个定义有些难以理解，下面的这张图能够帮助读者理解镜像的定义。


你可以在你的主机文件系统上找到有关这些层的文件。需要注意的是，在一个运行中的容器内部，这些层是不可见的。在我的主机上，我发现它们存在于/var/lib/docker/aufs目录下。

```bash
sudo tree -L 1 /var/lib/docker//var/lib/docker/
├── aufs
├── containers
├── graph
├── init
├── linkgraph.db
├── repositories-aufs
├── tmp
├── trust
└── volumes
7 directories, 2 files
```



## Container Definition

容器（container）的定义和镜像（image）几乎一模一样，也是一堆层的统一视角，唯一区别在于容器的最上面那一层是可读可写的。


要点：容器 = 镜像 + 读写层。并且容器的定义并没有提及是否要运行容器。

一个运行态容器（running container）被定义为一个可读写的统一文件系统加上隔离的进程空间和包含其中的进程。下面这张图片展示了一个运行中的容器。

[![4.png](http://dockone.io/uploads/article/20190626/1ccc2aa9e11e25e5ed2efd18cf1052c4.png)](http://dockone.io/uploads/article/20190626/1ccc2aa9e11e25e5ed2efd18cf1052c4.png)


正是文件系统隔离技术使得Docker成为了一个前途无量的技术。一个容器中的进程可能会对文件进行修改、删除、创建，这些改变都将作用于可读写层（read-write layer）。下面这张图展示了这个行为。

[![5.png](http://dockone.io/uploads/article/20190626/a20b70e3e4ca61faa2c3436e1bb2d93a.png)](http://dockone.io/uploads/article/20190626/a20b70e3e4ca61faa2c3436e1bb2d93a.png)


我们可以通过运行以下命令来验证我们上面所说的：

```bash
docker run ubuntu touch happiness.txt
```


即便是这个ubuntu容器不再运行，我们依旧能够在主机的文件系统上找到这个新文件。

```bash
find / -name happiness.txt/var/lib/docker/aufs/diff/860a7b...889/happiness.txt
```



## Image Layer Definition

为了将零星的数据整合起来，我们提出了镜像层（image layer）这个概念。下面的这张图描述了一个镜像层，通过图片我们能够发现一个层并不仅仅包含文件系统的改变，它还能包含了其他重要信息。





**Metadata Location：**我发现在我自己的主机上，镜像层（image layer）的元数据被保存在名为”json”的文件中，比如说：

```bash
/var/lib/docker/graph/e809f156dc985.../json
```


e809f156dc985...就是这层的id

现在，让我们结合上面提到的实现细节来理解Docker的命令。

[![9.png](http://dockone.io/uploads/article/20190626/062e7af0929dd205b2ac6efdd937d6f4.png)](http://dockone.io/uploads/article/20190626/062e7af0929dd205b2ac6efdd937d6f4.png)


docker create 命令为指定的镜像（image）添加了一个可读写层，构成了一个新的容器。注意，这个容器并没有运行。

[![10.png](http://dockone.io/uploads/article/20190626/162f02c6f9dce4ccdaa0c6c676da2196.png)](http://dockone.io/uploads/article/20190626/162f02c6f9dce4ccdaa0c6c676da2196.png)



### docker start <container-id>

[![11.png](http://dockone.io/uploads/article/20190626/1c38d4735e9760bdca025ff50a1b5386.png)](http://dockone.io/uploads/article/20190626/1c38d4735e9760bdca025ff50a1b5386.png)
Docker start命令为容器文件系统创建了一个进程隔离空间。注意，每一个容器只能够有一个进程隔离空间。

[![12.png](http://dockone.io/uploads/article/20190626/01aedf55bd21abbe607b3864d76f0ec0.png)](http://dockone.io/uploads/article/20190626/01aedf55bd21abbe607b3864d76f0ec0.png)


看到这个命令，读者通常会有一个疑问：docker start 和 docker run命令有什么区别。

[![13.png](http://dockone.io/uploads/article/20190626/275cc486d4ce7ecbdecf0ecc1de0a34b.png)](http://dockone.io/uploads/article/20190626/275cc486d4ce7ecbdecf0ecc1de0a34b.png)

从图片可以看出，docker run 命令先是利用镜像创建了一个容器，然后运行这个容器。这个命令非常的方便，并且隐藏了两个命令的细节，但从另一方面来看，这容易让用户产生误解。



### docker ps

[![14.png](http://dockone.io/uploads/article/20190626/f05a2a8dfc6641d8237306ff575aa283.png)](http://dockone.io/uploads/article/20190626/f05a2a8dfc6641d8237306ff575aa283.png)


docker ps 命令会列出所有运行中的容器。这隐藏了非运行态容器的存在，如果想要找出这些容器，我们需要使用下面这个命令。



### docker ps –a

[![15.png](http://dockone.io/uploads/article/20190626/d7f7b0ada7fc1c641c90745a959f9c05.png)](http://dockone.io/uploads/article/20190626/d7f7b0ada7fc1c641c90745a959f9c05.png)


docker ps –a命令会列出所有的容器，不管是运行的，还是停止的。



### docker images

[![16.png](http://dockone.io/uploads/article/20190626/d5b0b3e2e7acdcf35e7577d7670a46f7.png)](http://dockone.io/uploads/article/20190626/d5b0b3e2e7acdcf35e7577d7670a46f7.png)
docker images命令会列出了所有顶层（top-level）镜像。实际上，在这里我们没有办法区分一个镜像和一个只读层，所以我们提出了top-level镜像。只有创建容器时使用的镜像或者是直接pull下来的镜像能被称为顶层（top-level）镜像，并且每一个顶层镜像下面都隐藏了多个镜像层。



### docker images –a

[![17.png](http://dockone.io/uploads/article/20190626/ce083575e95a0e46b105c3596c12ca71.png)](http://dockone.io/uploads/article/20190626/ce083575e95a0e46b105c3596c12ca71.png)


docker images –a命令列出了所有的镜像，也可以说是列出了所有的可读层。如果你想要查看某一个image-id下的所有层，可以使用docker history来查看。



### docker stop <container-id>

[![18.png](http://dockone.io/uploads/article/20190626/a41de8fe542efe25e3620691ad9238df.png)](http://dockone.io/uploads/article/20190626/a41de8fe542efe25e3620691ad9238df.png)
docker stop命令会向运行中的容器发送一个SIGTERM的信号，然后停止所有的进程。



### docker kill <container-id>

[![19.png](http://dockone.io/uploads/article/20190626/ac1cbc31d4f191e26e05fb1a11f04d26.png)](http://dockone.io/uploads/article/20190626/ac1cbc31d4f191e26e05fb1a11f04d26.png)
docker kill 命令向所有运行在容器中的进程发送了一个不友好的SIGKILL信号。

[![20.png](http://dockone.io/uploads/article/20190626/b701a3c51e7d1915da3bea0bc43efcd1.png)](http://dockone.io/uploads/article/20190626/b701a3c51e7d1915da3bea0bc43efcd1.png)


docker stop和docker kill命令会发送UNIX的信号给运行中的进程，docker pause命令则不一样，它利用了cgroups的特性将运行中的进程空间暂停。具体的内部原理你可以在这里找到：[https://www.kernel.org/doc/Doc ... m.txt](https://www.kernel.org/doc/Documentation/cgroups/freezer-subsystem.txt)



### docker rm <container-id>

[![21.png](http://dockone.io/uploads/article/20190626/8fa2d6e19c29f18548624efd64eb6dfa.png)](http://dockone.io/uploads/article/20190626/8fa2d6e19c29f18548624efd64eb6dfa.png)

docker rm命令会移除构成容器的可读写层。注意，这个命令只能对非运行态容器执行。



### docker rmi <container-id>

[![22.png](http://dockone.io/uploads/article/20190626/32a7f413a5f8ff936dd0f4a31d25fdcc.png)](http://dockone.io/uploads/article/20190626/32a7f413a5f8ff936dd0f4a31d25fdcc.png)


docker rmi 命令会移除构成镜像的一个只读层。你只能够使用docker rmi来移除最顶层（top level layer）（也可以说是镜像），你也可以使用-f参数来强制删除中间的只读层。 

[![23.png](http://dockone.io/uploads/article/20190626/3c02ccf4e7a2a353af065d93b26ae89e.png)](http://dockone.io/uploads/article/20190626/3c02ccf4e7a2a353af065d93b26ae89e.png)



### docker commit <container-id>

docker commit命令将容器的可读写层转换为一个只读层，这样就把一个容器转换成了不可变的镜像。

[![24.png](http://dockone.io/uploads/article/20190626/28059b3a499faba896263c0ff077fe3a.png)](http://dockone.io/uploads/article/20190626/28059b3a499faba896263c0ff077fe3a.png)



### docker build

[![25.png](http://dockone.io/uploads/article/20190626/b22cd304f28c715ae3ae6812476b222d.png)](http://dockone.io/uploads/article/20190626/b22cd304f28c715ae3ae6812476b222d.png)
docker build命令非常有趣，它会反复的执行多个命令。

[![26.png](http://dockone.io/uploads/article/20190626/100712263ecf4544dd11602adc39ee3e.png)](http://dockone.io/uploads/article/20190626/100712263ecf4544dd11602adc39ee3e.png)


我们从上图可以看到，build命令根据Dockerfile文件中的FROM指令获取到镜像，然后重复地1）run（create和start）、2）修改、3）commit。在循环中的每一步都会生成一个新的层，因此许多新的层会被创建。

[![27.png](http://dockone.io/uploads/article/20190626/db64b3b38aff136d42e9ffeb81675bda.png)](http://dockone.io/uploads/article/20190626/db64b3b38aff136d42e9ffeb81675bda.png)



### docker exec 

命令会在运行中的容器执行一个新进程。

[![28.png](http://dockone.io/uploads/article/20190626/184f9d55770ca036cb5b1e6d96ce4a12.png)](http://dockone.io/uploads/article/20190626/184f9d55770ca036cb5b1e6d96ce4a12.png)



### docker inspect

命令会提取出容器或者镜像最顶层的元数据。

[![29.png](http://dockone.io/uploads/article/20190626/70cdbaf975c88bc83423d88be85476b5.png)](http://dockone.io/uploads/article/20190626/70cdbaf975c88bc83423d88be85476b5.png)



### docker save

命令会创建一个镜像的压缩文件，这个文件能够在另外一个主机的Docker上使用。和export命令不同，这个命令为每一个层都保存了它们的元数据。这个命令只能对镜像生效。

[![30.png](http://dockone.io/uploads/article/20190626/1714c3dd524c807bf9c9b4d0fbe4d056.png)](http://dockone.io/uploads/article/20190626/1714c3dd524c807bf9c9b4d0fbe4d056.png)



### docker export

命令创建一个tar文件，并且移除了元数据和不必要的层，将多个层整合成了一个层，只保存了当前统一视角看到的内容（译者注：expoxt后的容器再import到Docker中，通过docker images –tree命令只能看到一个镜像；而save后的镜像则不同，它能够看到这个镜像的历史镜像）。

[![31.png](http://dockone.io/uploads/article/20190626/b513dd5467f23fdd23523f60242d5dcb.png)](http://dockone.io/uploads/article/20190626/b513dd5467f23fdd23523f60242d5dcb.png)



### docker history

命令递归地输出指定镜像的历史镜像。
