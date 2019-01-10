---
layout:     post
title:    轻松构建微服务之高效发布
no-post-nav: true
category: other
tags: [arch]
excerpt: Devops 持续集成 蓝绿发布
---

## docker

我们先来了解下docker的原理,如何才能制造出一个真正隔离的软件运行环境.

### namespace

docker在创建容器进程的时候可以指定一组namespace参数，这样容器就只能看到当前namespace所限定的资源，文件，设备，网络。用户，配置信息，而对于宿主机和其他不相关的程序就看不到了，PID namespace让进程只看到当前namespace内的进程，Mount namespace让进程只看到当前namespace内的挂载点信息，Network namespace让进程只看到当前namespace内的网卡和配置信息，

docker利用namespace机制隔离出一个软件执行环境,我们可以用下图来将docker和虚拟机技术做一个对比

![](https://pigpdong.github.io/assets/images/2019/docker/dockervm.jpg)

一个centOS的KVM启动起来后,即使什么不做也需要消耗200M的内存,而且用户进程运行在虚拟机里,对宿主机操作系统的调用不可避免会受到虚拟化软件的拦截,而容器化的应用依然是宿主机上的一个普通进程.

###  cgroup

全名 linux control group，用来限制一个进程组能够使用的资源上限，如CPU，内存，网络等，另外Cgroup还能够对进程设置优先级和将进程挂起和恢复，cgroup对用户暴露的接口是一个文件系统，/sys/fs/cgroup下 这个目录下面有 cpuset,memery等文件，每一个可以被管理的资源都会有一个文件，如何对一个进程设置资源访问上限呢？在/sys/fs/cgroup目录下新建一个文件夹，系统会默认创建上面一系列文件，然后docker容器启动后，将进程ID写入taskid文件中，在根据docker启动时候传人的参数修改对应的资源文件

```
$ docker run -it --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash
```

Linux Cgroup的设计还是比较易用的,简单理解就是一个子系统目录加上一组资源限制文件,而对docker等linux容器项目而言,只需要在每个子系统下面,为每个容器创建一个控制组也就是一个文件夹,让后在容器启动后写入进程PID.

### 文件系统

即使开启了Mount Namespace,容器进程看到的文件系统还是和宿主机一样的,mount namespace修改的时容器进程对挂载点的认知,他对容器进程视图的改变,一定伴随挂载操作才生效,但是作为一个普通用户,我们希望在容器内看到的文件系统就是一个独立的隔离环境,而不是继承自宿主机上的文件系统.

我们可以在容器启动后马上挂载他的整个根目录,这个挂载对宿主机不可见,然后通过chroot来更改change root file system更改进程的根目录到挂载的位置，一般会通过chroot挂载一个完整的linux的文件系统,但是不包括linux内核，这样当我们交付一个docker镜像的时候不仅包含需要运行的程序还包括这个程序依赖运行的这个环境，因为我们打包了整个依赖的linux文件系统，对一个应用来说，操作系统才是他所依赖的最完整的依赖库，

这个挂载在容器根目录下,用来为容器进程提供隔离后的执行环境的文件系统,就是所谓的容器镜像,它还有一个专业的名词rootFS

由于rootfs里面打包的不仅仅是用户程序,而是整个系统的文件目录,也就是是应用和应用依赖的类库和环境变量都在里面.

### 增量层

docker在镜像的设计中引入层的概念，也就是用户在制作docker镜像中的每一次修改都是在原来的rootfs上新增一层roofs,之后通过一种联合文件系统union fs的技术进行合并，合并的过程中如果两个rootfs中有相同的文件则会用最外层的文件覆盖原来的文件来进行去重操作，举个例子，我们从镜像中心pull一个mysql的镜像到本地，当我们通过这个镜像创建一个容器的时候，就在这个镜像原有的层上新加了一个增roofs,这个文件系统只保留增量修改，包括文件的新增删除，修改，这个增量层会借助union fs和原有层一起挂载到同一个目录，这个增加的层可以读写，原有的其他层只能读，这样保证了所有对docker镜像的操作都是增量，之后用户可以commit这个镜像将对这个镜像的修改生成一个新的镜像，新的镜像就包含了原有的层和新增的层，只有最原始的层才是一个完整的linux fs, 那么既然只读层不允许修改，那么我怎么删除只读层的文件呢，这个时候只需要在读写层也就是最外层生成一个whiteout文件来遮挡原来的文件就可以了，

![](https://pigpdong.github.io/assets/images/2019/docker/rootfs.png)

```
$ docker image inspect ubuntu:latest
...
     "RootFS": {
      "Type": "layers",
      "Layers": [
        "sha256:f49017d4d5ce9c0f544c...",
        "sha256:8f2b771487e9d6354080...",
        "sha256:ccd4d61916aaa2159429...",
        "sha256:c01d74f99de40e097c73...",
        "sha256:268a067217b5fe78e000..."
      ]
    }
```

###

dockerfile 可以通过docfile生成一个镜像，docfile里面可以指定 from的原始镜像，以及自定义操作例如拷贝文件等，容器启动命令等

例如下面的dockerfile,FROM原语指定了原始镜像是python:2.7-slim,避免了先从一个原始的ubantu镜像,然后通过apt-get install按照python

WORDIR 切换工作目录

ADD拷贝文件

RUN 在容器里执行shell

CMD 指定容器进程,相当于 docker rum python app.py

```
# 使用官方提供的 Python 开发镜像作为基础镜像
FROM python:2.7-slim

# 将工作目录切换为 /app
WORKDIR /app

# 将当前目录下的所有内容复制到 /app 下
ADD . /app

# 使用 pip 命令安装这个应用所需要的依赖
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# 允许外界访问容器的 80 端口
EXPOSE 80

# 设置环境变量
ENV NAME World

# 设置容器进程为：python app.py，即：这个 Python 应用的启动命令
CMD ["python", "app.py"]

```


## kubernets

一个做容器编排和调度的工具，kubernets的最小调度单元是POD，一个POD可以管理一组同生命周期的容器，k8s提供一个restful的客户端api供用户使用，所以会有一个APIserver来接受请求，通过etcd作为数据库来存储请求中得CRUD操作，而其他模块例如控制器中的调度单元，会扫描数据库中的记录，如果有新的POD还没有分配物理节点，则会执行调度动作，如果发现新增了副本数量，就会增加POD副本，如果修改了POD相关配置就去执行，而每一个节点上面都会允许一个kube-proxy用来接收外部请求后转发，而docker采用的插拔容器，可以使用docker引擎，也可以用其他的引擎。


![](https://pigpdong.github.io/assets/images/2019/docker/k8s.png)

## Devops

我们如何利用k8s做到自动化打包和发布

- 1.创建pileline 指定项目名称和对应的tag,以及依赖工程
- 2.根据项目名称和tag去gitlab上拉取最新的代码(shell 脚本 Runtime执行脚本)
- 3.利用maven进行打包,这个时候可以为maven创建一个单独的workspace(shell 脚本)
- 4.根据预先写好的docfile,生成镜像,并上传镜像 (shell 脚本)
- 5.通过k8s的api在测试环境发布升级
- 6.通过灰度等方案发布到生产环境
