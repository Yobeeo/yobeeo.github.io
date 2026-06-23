---
layout: post
title: "Docker 入门与实践"
date: 2021-12-21 00:00:00 +0800
categories: DevOps
tags: [Docker, 容器, 虚拟化]
---
## 一. 虚拟化技术分类

### 1.1 主机级虚拟化

虚拟化整个物理硬件平台，如VMware 

* **Type-1**

  宿主机+虚拟机管理器（VMM）+虚拟机

* **Type-2**: VMware  Virtual Box


  主机级虚拟化需要安装内核，内核是不可缺少的，用于资源调度，但真正产生生产力的是在内核上安装的其他应用；

  虚拟机隔离的是内核空间，会经过两级调度，有一定的资源浪费

  在创建虚拟机时就做了资源限制

### 1.2 容器级虚拟化

## 二. 容器

容器是一种技术，用于隔离同一系统中的不同进程，最早出现于Free BSD上，叫做jail，目的是为了进程的安全运行（一个进程的行为不会影响到容器外其他进程）；jail后应用到linux上，叫做vserver，当时主要是使用`chroot()`实现

### 2.1 namespaces

| namespace | 系统调用参数  | 隔离内容                   | 内核版本 |
| --------- | ------------- | -------------------------- | -------- |
| UTS       | CLONE_NEWUTS  | 主机名和域名               | 2.6.19   |
| IPC       | CLONE_NEWIPC  | 信号量、消息队列和共享内存 | 2.6.19   |
| PID       | CLONE_NEWPID  | 进程编号                   | 2.6.24   |
| Network   | CLONE_NEWNET  | 网络设备、网络栈、端口等   | 2.6.29   |
| Mount     | CLONE_NEWNS   | 挂载点（文件系统）         | 2.4.19   |
| User      | CLONE_NEWUSER | 用户和用户组               | 3.8      |

上述每种资源在内核中，都能切分为多个互相隔离的部分，每个部分都是一个名称空间（namespace）。

linux通过namespace原生支持上述资源的切分，以系统调用的形式提供接口

* `clone()`  创建子进程
* `setns()`  设置名称空间 
* `share()` `unshare()`  从名称空间中添加、移除

### 2.2 cgroups

`control groups（cgroups）`是另一种技术，能够把系统资源按组划分，然后应用到不同的namespace中。能够划分的资源有：

* 块设备IO（blkio）
* CPU
* CPU资源使用报告（cpuacct）
* 多处理器平台上的CPU集合（cpuset）
* 设备访问，比如光驱只能给一个namespace用（devices）
* 挂起或恢复任务（freezer）
* 内存用量报告（memory）
* 对cgroups中的任务进行统一性能测试（perf_event）
* cgroups中的任务创建的数据报文的类别标识符（net_cls）

### 2.3 发展

* LXC（Linux Container），一组工具（命令脚本和template），用于在linux上自动安装生成所要的隔离环境，同一环境可以使用门槛挺高，数据难迁移

* Docker，可理解为LXC的增强版，通过镜像技术，简化了LXC。
* OCI标准，旨在围绕容器格式和运行时制定一个开放式标准
* OCF标准，open container format, runC为其实现

## 三. docker架构

docker是一门容器技术，解决软件跨环境迁移的问题

docker可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的Linux机器上；docker是完全采用沙箱机制，相互隔离；容器性能开销极低

### 3.1 C/S架构

docker是C/S结构，C/S端都由docker一个程序提供，由其不同的子程序实现不同功能。一般有3个组成部分：

* **registry**  统一存放镜像的地方，默认位置是docker hub，国内有其备份镜像

  一个registry通常有两部分组成：Repository和Index

  * Repository：一般一个程序都有一个自己的Repository，存放不同版本的镜像，通过不同的tag进行区分。Repository可分为顶级仓库和用户仓库（latest标签，指向最新版镜像；stable标签，指向最新的稳定版镜像）

  * Index：维护用户账户、镜像的校验信息以及公共命名空间的信息，可用于`docker search`搜索

* **docker host**，由docker daemon子程序运行的守护进程，监听在本地的unix socket，只允许本地访问（为了安全）

  其中有两个重要部分：Containers和Images。

* **client**  接收用户输入命令

client到host，host到registry，都是通过http协议通信，使用标准的restful接口

### 3.2 镜像架构

docker镜像含有启动容器需要的文件系统及其他内容，实际是一种分层的文件系统，最底层为`bootfs`（一般使用宿主机的bootfs，因此启动容器很快）；然后为`rootfs`，又称为base image；再往上可以叠加其他的镜像文件

* `bootfs`  用于系统引导的文件系统，包含bootloader(引导加载程序)和kernel(内核)，容器启动完成后就会被卸载（从内存中移除）以节约内存资源

* `rootfs`  表现为容器的根文件系统，包含的就是linux系统中的/dev , /bin等标准目录和文件

  传统模式中，系统启动时，内核先以`只读`模式挂载rootfs，待完整性自检完成后，再以`读写`模式重新挂载

  而docker中，rootfs只以`只读`模式挂载，而后通过“联合挂载”的技术额外挂载一个可写层

不同的linux发布版本，bootfs基本一样，而rootfs不同，如ubuntu，centos等

![][markdowns/pics/devOps/docker/docker-image.png]

## 四. docker使用

### 4.1 安装docker

以centos为例，国内可能需要软件源，可使用清华的源，在`/etc/yum.repos.d/`中添加对应文件

```shell
# yum install -y docker

1、更新yum源
    yum update		
2、安装依赖包
    yum install -y yum-utils device-mapper-persistent-data lvm2
3、设置yum源
4、yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
5、安装docker
    yum install -y docker-ce
6、验证是否安装成功
    docker -v
```

### 4.2 镜像加速器

docker配置文件默认为`/etc/docker/daemon.json`，为了镜像下载快速，国内可能需要配置镜像加速器，比如清华的

```shell
{
	"registry-mirrors":[
		"https://registry.docker-cn.com"
	]
}
```

### 4.3 docker命令

#### 4.3.1 docker服务命令

```shell
systemctl start docker[.service]
systemctl stop docker[.service]
systemctl restart docker[.service]
systemctl status docker[.service]
systemctl enable docker[.service]		# 开机启动docker
```

#### 4.3.2 查看帮助

docker命令是分组的，也建议使分组的命令，但是也兼容之前未分组的命令

```shell
docker [cmd] --help
# docker --help
# docker container --help

# docker的命令组包括
  builder     # Manage builds
  config      # Manage Docker configs
  container   # Manage containers
  context     # Manage contexts
  engine      # Manage the docker engine
  image       # Manage images
  network     # Manage networks
  node        # Manage Swarm nodes
  plugin      # Manage plugins
  secret      # Manage Docker secrets
  service     # Manage services
  stack       # Manage Docker stacks
  swarm       # Manage Swarm
  system      # Manage Docker
  trust       # Manage trust on Docker images
  volume      # Manage volumes
```

#### 4.3.3 查看version

```shell
docker version   # 查看C/S各自的版本信息
docker info      # 更详细的docker信息
```

#### 4.3.4 常用操作

##### 1. 镜像相关 image

*  `docker [image] COMMAND`，`COMMAND`可以为
  *  `build`       Build an image from a Dockerfile
  *  `history`     Show the history of an image
  *  `import`      Import the contents from a tarball to create a filesystem image
  *  `inspect`     Display detailed information on one or more images
  *  `load`        Load an image from a tar archive or STDIN
  *  `ls`            List images，或`docker images`，ImageID字段默认只显示一部分，可使用`--no-trunc`参数
  *  `prune`       Remove unused images
  *  `pull`        Pull an image or a repository from a registry，或`docker pull`
  *  `push`        Push an image or a repository to a registry，或`docker push`
  *  `rm`          Remove one or more images，或`docker rmi` 
  *  `save`        Save one or more images to a tar archive (streamed to STDOUT by default)，或`docker save`
  *  `tag`         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE

* `docker search [OPTIONS] TERM`  查找镜像

```shell
 #查看
     docker images				
​ #搜索
     docker search image_name			
​ #拉取(无版本号拉取最新版本)
     docker pull image_name[:version]		 #hub.docker.com 上可查看有哪些镜像及版本
​ #删除
     docker rmi image_id  或 docker rmi image_name:version    
​ #删除所有镜像
     docker rmi ` docker images -q`		
```

##### 2. 容器相关 container

* `docker [container] COMMAND`，`COMMAND`可以为
  *  `attach`      Attach local standard input, output, and error streams to a running container
  *  `commit`     Create a new image from a container's changes
  *  `cp`          Copy files/folders between a container and the local filesystem 
  *  `create`      Create a new container
  *  `diff`        Inspect changes to files or directories on a container's filesystem
  *  `exec`        Run a command in a running container
  *  `export`      Export a container's filesystem as a tar archive
  *  `inspect`     Display detailed information on one or more containers
  *  `kill`        Kill one or more running containers
  *  `logs`        Fetch the logs of a container
  *  `ls`          List containers，或 `docker ps` ，默认只显示在运行的container，`-a`参数查看所有container
  *  `pause`       Pause all processes within one or more containers
  *  `port`        List port mappings or a specific mapping for the container
  *  `prune`       Remove all stopped containers
  *  `rename`      Rename a container
  *  `restart`     Restart one or more containers
  *  `rm`          Remove one or more containers
  *  `run`         Run a command in a new container
  *  `start`       Start one or more stopped containers
  *  `stats`       Display a live stream of container(s) resource usage statistics
  *  `stop`        Stop one or more running containers
  *  `top`         Display the running processes of a container
  *  `unpause`     Unpause all processes within one or more containers
  *  `update`      Update configuration of one or more containers
  *  `wait `       Block until one or more containers stop, then print their exit codes

```shell
 #查看
     docker ps [-a]         默认查看打开的容器，-a查看所有
​ #创建
docker run -it –name=redis redis:7 /bin/bash
     -i 保持一直运行
     -t 给容器分配一个伪终端（使用该命令，会自动进入容器，exit退出后容器就关闭了）
     -d 以守护（后台）模式创建容器
     -p 1234：5678 端口映射，容器外1234映射到容器内5678
     --name 新建的容器名
     -e MACRO_NAME = MACRO_VALUE   指定环境变量
     image_name:version
     /bin/bash      进入容器初始化执行的命令，默认为`bin/bash`
     --rm 退出容器后则马上删除容器
     --network netname  指定网络
​ #进入
     docker exec -it container_name /bin/bash
​ #启动
     docker start container_name
​ #停止
     docker stop container_name
​ #删除
     docker rm container_name or container_id
​ #查看信息
     docker inspect container_name
```

##### 3. 网络相关

创建容器时(`create`或`run`)可指定容器所在的网络属性

* network

  使用`--network` 参数，不指定默认为docker0网络

  ```shell
  # 查看网络
  docker network ls   
  
  docker run -it --name r1 --network bridge -it --rm busybox:latest /bin/bash
  ```

* hostname

  `hostname`  查看主机名，容器的主机名默认和container id一样，也可以在创建容器时使用`-h`指定

  ```shell
  docker run -it --name r1 --network bridge -h howe.test.com -it --rm busybox:latest /bin/bash
  ```

  如果需要通过主机名访问其他主机，需要通过域名解析，或者使用`/etc/host`，使用`--add-hosts host:ip`可在docker中注入host文件

  

  `/etc/resolv.conf`文件记录了DNS服务器的地址，域名解析时会在这里面找DNS服务器

  `nslookup -type=A www.baidu.com`   解析网址

* dns

  dns也可以在创建容器时指定，使用`--dns`参数

  ```shell
  docker run -it --name r1 --network bridge -h howe.test.com --dns 114.114.114.114 -it --rm busybox:latest /bin/bash
  ```

  

  还可以指定dns-search列表（当直接输入名称时，名称会和dns-search列表中的拼接，成为一个完整的域名[https://www.bilibili.com/video/av843288923]）

  ```shell
  docker run -it --name r1 --network bridge -h howe.test.com --dns 114.114.114.114 --dns-search linux.io -it --rm busybox:latest /bin/bash
  ```

  

##### 其他









### 4.4 制作镜像

#### 4.4.1 容器转镜像

docker commit +  docker tag + docker push / docker pull

#### 4.4.2 容器转压缩包

docker save + docker load

## 五. docker网络

### 5.1 docker网络类型

`docker network ls`    查看当前有哪些网络，默认有3种：bridge、host、none

#### 5.1.1 bridge

容器启动，默认使用bridge网络。默认是NAT bridge，即docker0那个网络（既能当交换机，又能当网卡）

linux可以用软件模拟网卡，一次创建一对网卡，一半在容器里，另一半在宿主机（或虚拟路由）上，即实现了容器和外部的相连

* `ifconfig`  查看本机网卡，其中`docker0`网卡是docker默认创建的；`vethb69e57a`是启动的容器创建的，是一对，此处显示的是在宿主机上的一半，另一半在容器中

  ```shell
  [root@VM-245-18-centos ~]# ifconfig    
  ...
  vethb69e57a: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
          ether 2a:ff:e8:9f:04:0b  txqueuelen 0  (Ethernet)
          RX packets 0  bytes 0 (0.0 B)
          RX errors 0  dropped 0  overruns 0  frame 0
          TX packets 0  bytes 0 (0.0 B)
          TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
  ```
  
* `yum install -y bridge-utils
  brctl show`  查看网卡的接口情况，如下图所示，即`vethb69e57a`插在`docker0`上的

  ```shell
  [root@VM-245-18-centos ~]# brctl show
  bridge name     bridge id               STP enabled     interfaces
  docker0         8000.02428e64bb24       no              vethb69e57a
  ```

  
  
* `iptables -t nat -vnL`  查看路由表，包括NAT伪装

  ```shell
  [root@VM-245-18-centos ~]# iptables -t nat -vnL
  ...
  Chain POSTROUTING (policy ACCEPT 9128 packets, 1068K bytes)
   pkts bytes target     prot opt in     out     source               destination         
      0     0 MASQUERADE  all  --  *      !docker0  192.168.10.0/24      0.0.0.0/0           
  
  ```

#### 5.1.2 host

一般来说一个容器有6种独立的namespace，但是可以仅使`User`、`Mount`、`Pid`三种namespace独立，而另外三种`UTS`、`Net`、`IPC`共享，即对外使用同一个主机名、IP地址，此时就是联盟式网络(joined network)

更特别的，共享的`UTS`、`Net`、`IPC`可以是主机的，此时就是host网络

#### 5.1.3 none

不适用网络，相当于没有网卡，只有本地环回lo

#### 5.1.4  网络架构模拟

使用`ip`命令可模拟容器间的网络架构

```shell
rpm -q iproute  # 查看是否安装 ip命令
yum install iptable -y  # 安装ip工具
ip netns help   # 网络名称空间的命令，使用ip管理网络名称空间时，只有网络名称空间是隔离的，其他名称空间都是共享的
ip link help    # 虚拟网卡对的命令
```



* `ip netns list`

* `ip netns add ns1`  添加ns1的网络名称空间

* `ip netns exec name ifconfig -a`  在name网络名称空间中执行命令



* `ip link show` 显示虚拟网卡对

* `ip link add name net1.1 type veth peer name net1.2`  创建虚拟网卡对，默认未激活，使用ifconfig不可见，此时使用`ip link show`可以看见

* `ip link set dev net1.1 netns ns1`  把net1.1设置到网络名称空间ns1中，`ip link show`只能看到一个了，另一个可以在ns1中使用`ifconfig`查看

### 5.2 与容器内服务通信

容器所在主机外的其他主机，无法直接访问容器，因为是在NAT后面，需要配置路由或者配置DNAT规则

#### 5.2.1 配置路由

* 在宿主机上配置路由，到达某个目的容器地址经过某个地址，但是不方便，因为可能需要配置很多条目

#### 5.2.2 开放端口

* 开放端口(即指定NAT规则)，创建容器时指定`-p`参数开放端口，其他主机访问容器时，访问宿主机的这个端口即可
  * `-p {containerPort}`  将容器内的端口映射到宿主机的动态端口，如果物宿主机有多个IP，则会将所有IP的这个端口映射到容器的指定端口

    ```shell
    [root@VM-245-18-centos ~]# docker port porttest
    80/tcp -> 0.0.0.0:32770
    ```

  * `-p {hostPort}:{containerPort}`  将容器内的指定端口映射到宿主机所有可用IP的指定端口

  * `-p {hostIp}::{containerPort}`  将宿主机指定IP的动态端口，映射到容器的指定端口

    ```shell
    [root@VM-245-18-centos ~]# docker port porttest
    80/tcp -> 9.208.245.18:32769
    ```

  * `-p {hostIp:hostPort}:{containerPort}`  将容器内的指定端口映射到宿主机指定IP的指定端口

 `-p`选项可指定多次，以暴露多个端口

 `-P` 若在镜像中指定了开放某个端口，基于镜像启动容器端口是默认不暴露的，除非使用`-P`选项

 配置了NAT规则，可使用`iptables -t nat -vnL` 查看，也可以使用`docker port {containerName}` 查看



### 5.3 设置容器的网络

#### 5.3.1 设置网络类型

创建容器使用`--network`参数时，不指定默认为`bridge`

* `none`  不设置网络

* `host`  为共享宿主机网络名称空间
* `container {containerName}`  为共享containerName的网络名称空间，即联盟式网络

#### 5.3.2 创建自定义网络

`docker network create` 可创建自定义的网络（`docker info` 查看Plugins，为docker支持的插件）

#### 5.3.3 修改docker0属性

在配置文件`/etc/docker/daemon.json`文件中可修改docker0桥网络的属性，修改后重启docker服务生效

```shell
{
  "bip":"192.168.1.5/24",	#bridge ip的简写，即docker0 桥的IP地址和子网掩码
  "mtu":1500,
  "default-gateway":"192.168.1.1",
  "dns":["8.8.8.8","4.4.4.4"]
}
```

#### 5.3.4 远程控制docker

docker默认只监听本地的`socket`，不支持远程控制。若要使用远程控制，需修改配置文件

```shell
{
  "hosts":["tcp://0.0.0.2357", # 开放的远程链接
           "unix:///var/run/docker.sock" # 默认的本地sock文件
  ]
}
```

重启docker服务后，docker客户端命令即可指定`-H`或`--host`参数，连接到指定docker服务器，远程进行操作



## 六. 存储卷

docker镜像由多个只读层叠加而成，启动容器时，docker会加载只读层并在镜像顶层添加一个读写层

如果运行中的容器修改了现有的文件，该文件将会从读写层下面的只读层复制到读写层，该文件的只读版本仍然存在，只是被读写层中的该文件的副本隐藏了

如果直接在联合文件系统上修改文件，效率慢，容器间共享不方便，而且删除容器后数据会丢失（关闭、重启不会丢失），存储卷的概念由此而来

### 6.1 存储卷概念

存储卷是宿主机中的一个目录或文件，用于宿主机和容器的目录映射，可作为容器内数据的持久化

### 6.2 存储卷使用

启动或创建容器时，指定存储卷

#### 6.2.1 docker-managed volume

* `-v {containerDir}`    只指定容器内的挂载点，宿主机上将有docker自动分配一个目录

#### 6.2.2 bind-mount volume

* `-v {hostDir}:{containerDir}`    宿主机的hostDir映射到容器内的containerDir
* `--volumes-from {containerName}`    复制containerName的存储卷设置



建立存储卷后，可使用`docker inspect {containerName}`查看`Mounts`节点观察，其他注意点：

* 目录必须是绝对路径

* 如果目录不存在，会自动创建

* 可以挂载多个数据卷，用多-v 参数即可



docker inspect查看信息时，可使用go模板

```shell
docker inspect -f {{.Mounts}} containerName
```

## 七. Dockerfile
Dockerfile是一个文本文件，也可用于制作镜像(执行`docker build`，镜像制作好后启动镜像，使用`docker run`)

Dockerfile应该尽量简洁，因为一条命令就会覆盖一层，会导致镜像增大

### 7.1 Dockerfile注意事项

* Dockerfile文件名首字母必须大写，该文件所在目录即为build目录，Dockerfile只能引用build目录之内的文件（包括子目录）
  `.dockerignore`文件，用于指定不包含的文件

* 注释使用`#`开头，指令不区分大小写（建议大写）
* 指令一般一行一条，从上往下顺序执行；第一个非注释行必须为`FROM`指令，以指定基础镜像

### 7.2 Dockerfile指令

#### 7.2.1 FROM  

`FROM`须为Dockerfile文件的**第一个非注释行**

```dockerfile
FROM [{registry}]{repository}[{:tag}]
# 或
FROM [{registry}]{repository}@{digest}
```
未指定registry时，`docker build`会在当前docker主机上查找指定的镜像文件，不存在时再从docker hub上拉取，若没有则报错

一个仓库可能有多个重名镜像，这时候可使用digest确定使用哪个镜像

#### 7.2.2 MAINTAINER(已废弃)

`MAINTAINER`指定Dockerfile作者的信息(已使用`LABLE`代替)

```dockerfile
MAINTAINER "detail"
# 如
MAINTAINER "{name} <{email}>"
```
#### 7.2.3 LABEL
`LABEL`设置`{key}={value}`样式的元数据，可替代`MAINTAINER`

```dockerfile
LABEL maintainer="howebai <howe_bai@163.com>"
```

#### 7.2.4 COPY

`COPY`从docker主机复制文件到创建的新镜像文件

```dockerfile
COPY src dest
# 或
COPY ["src1","src2",... "dest"]
```
复制准则
* src必须为Dockfile build目录内的路径，不能是其父目录中的路径
* 若src是目录，复制的是目录下的所有文件，src本身不复制
* 若文件名中有空格，必须使用第二种有引号的形式
* dest是目录时，必须以`/`结尾；dest不存在时会自动创建
#### 7.2.5 ADD

`ADD`类似`COPY`指令，但另外还支持tar文件展开和URL路径

```dockerfile
ADD src dest
# 或
ADD ["src1","src2",... "dest"]
```
准则
* 和`COPY`相同
* 若src为URL，URL指定的文件会下载下来
  dest不以`/`结尾，直接创建为dest
  dest以`/`结尾，保存为dest/files
* 若src为本地系统的压缩格式tar文件，将会自动展开为目录，相当于执行`tar -x`（URL格式的tar文件不会自动展开）

#### 7.2.6 WORKDIR

`WORKDIR`用于指定容器内的工作目录，可以设置多次，有用到相对路径时，向上找最近的`WORKDIR`

```dockerfile
WORKDIR {dir}
```

#### 7.2.7 VOLUME

`VOLUME`指定存储卷，只能指定docker-managed volume（只能指定镜像中的路径）

```dockerfile
VOLUME {containerDir}
# 或
VOLUME ["containerDir"]
```

#### 7.2.8 EXPOSE

`EXPOSE`指定待开放端口，相当于容器启动时的`-p port`命令（只能动态绑定，容器内端口映射为宿主机的随机端口）

安全原因，写在Dockerfile中的端口不会直接暴露出去，需要在启动时使用`-P`选项

```dockerfile
EXPOSE {port1}[/{protocol1}] {port2}[/protocol2]
# 如
EXPOSE 11211/tcp 11211/udp
```

protocol为tcp或udp，默认tcp

#### 7.2.9 ENV

`ENV`为镜像定义所需要的环境变量，并可被Dockerfile中位于其后的其他命令使用，引用变量格式与shell一致

```dockerfile
ENV {key} {value}
# 或
ENV {key1}={value1} {key2}={value2} ...
```

* 第一种格式，key后的所有内容都会被视为value的组成部分，因此一次只能设置一个变量
* 第二种格式可用于设置多个变量，若变量中包含空格可使用引号(`""`)或者反斜线(`\`)进行转义，另外`\`也可以用于续行

若启动容器时(`docker run`阶段)指定了`-e`参数，使用了同名的环境变量，则以`-e`指定的为准（`-e`只是修改了环境变量的值，并没有修改通过Dockerfile生成的镜像内容(`docker build`阶段)）

#### 7.2.10 RUN

`RUN`指定**在`docker build`过程中**要执行的命令，通常用于安装应用和软件包

```dockerfile
RUN {shellCommand}
# 或
RUN ["excutable", "param1","param2",...]
```

第一种格式通常为shell命令，系统默认以`/bin/sh -c`来运行它（先启动shell，再传递给shell当做参数），因此该进程在容器中的PID不为1（shell中调用`exec`除外），且不能接收Unix信号（docker stop停止容器时，不能接收到SIGTERM信号）

第二种格式不会以`/bin/sh -c` 执行，因此shell的操作不能使用（如变量替换、通配符`*?`等），当然，如果想使用shell调用，也可以这样做

```dockerfile
RUN ["/bin/bash", "-c", "excutable", "param1",...]
```

#### 7.2.11 CMD

`CMD`指定**在`docker run`过程中**要执行的命令

* `docker run`时可传入命令，替换`CMD`，否则直接使用`CMD`
* 若指定多个`CMD`，只有最后一个生效

```dockerfile
CMD {shellCommand}
# 或
CMD ["excutable", "param1","param2",...]
# 或(这种需要搭配  ENTRYPOINT 使用)
CMD ["param1", "param2",...]
```

前两种与`RUN`类似

第三种格式需搭配`ENTRYPOINT`使用，相当于后者的参数

#### 7.2.12 ENTRYPOINT

`ENTRYPOINT`用于为容器指定默认的运行程序，另外`docker run`时也可以使用`--entrypoint`选项指定

* docker run命令传入的命令会覆盖`CMD`指令，都将被当做`ENTRYPOINT`的参数
* `ENTRYPOINT`也可以指定多个，但只有最后一个生效

```dockerfile
ENTRYPOINT {command}
# 或
ENTRYPOINT ["excutable", "param1", "param2",...]
```



一般使用脚本方式生成容器内服务的配置，并在脚本内调用`CMD`，由`ENTRYPOINT`调用脚本

#### 7.2.13 USER

`USER`用于指定运行Dockerfile和运行容器时（指定执行`CMD` `RUN` `ENTRYPOINT`时的用户），执行各项命令的用户名（或UID），默认情况使用的root用户

```dockerfile
user {uid}|{username}
```

需要注意的是，该用户需要用一定的权限，否则执行命令时可能会失败，一般为`/etc/passwd`中的用户

#### 7.2.14 HEALTHCHECK

`HEALTHCHECK`用于按自定方式对容器进行健康检查

```dockerfile
HEALTHCHECK [options] CMD command
HEALTHCHECK NONE
# 例子 成功什么都不做，失败返回1
HEALTHCHECK --interval=5m --timeout=3s\
    CMD curl -f http://localhost/ || exit 1
```

第一种方式，在容器内运行指定的命令进行健康检查

第二种方式，禁用健康检查（即时从基础镜像中继承的有也禁用）

**options**

* `--interval={duration}`    检查间隔时间，默认30s
* `--timeout={duration}`    每次检查超时时间，默认30s
* `--start-period={duration}`    开始检查延迟时间，默认0s（即容器拉起后多长时间开始检查，一般生产服务会慢于容器拉起时间）
* `--retries={times}`    重试多少次表示检查失败，默认3次

**检查结果**

* `0`    成功，容器健康
* `1`    失败，容器不健康
* `2`    保留的，不使用

#### 7.2.15 SHELL

`SHELL`用于指定运行Dockerfile或容器运行时的shell（指定执行`CMD` `RUN` `ENTRYPOINT`时的shell），linux默认`["/bin/sh","-c"]`，windows默认`["cmd","/S","/C"]`

```dockerfile
SHELL ["excutable","param1",...]
```

可指定多次，但只有最后一次生效

#### 7.2.16 STOPSIGNAL

`STOPSIGNAL`用于设置停止容器的系统信号。

`docker stop`命令会产生一个15的信号，容器内进程id为1的进程为主进程，能够接受到这个信号并停止容器

```dockerfile
STOPSIGNAL {signal_num}|{signal_name}
# 如
STOPSIGNAL 9
# 或者
STOPSIGNAL SIGKILL
```

#### 7.2.17 ARG

`ARG`用于在`docker build`过程中传递参数，使用`--build-arg`选项

```dockerfile
ARG arg_name[=default_value]
```

如果有同名`ENV`，那么`ENV`始终覆盖`ARG`

#### 7.2.18 ONBUILD

`ONBUILD`用于在Dockerfile中指定一个触发器，当存在`ONBUILD`关键字的镜像作为基础镜像的时候，当执行完`FROM`后就会执行`ONBUILD`的命令

```dockerfile
ONBUILD other_instructions
```

**注意**

* `ONBUILD`后的指令不能为`ONBUILD`、`FROM`、`MAITAINER`
* 使用`ONBUILD`的镜像应该使用特殊命名，如: ruby2.0-onbuild
* 另外指令使用COPY或ADD时，若当前宿主机上没有对应文件会报错

### 7.3 Dockerfile构建

```shell
docker build -f dockerfile -t newImageName:version ./
```

`-f` 指定dockerfile

`-t` 指定新镜像名（镜像名:版本号）

`./` 指定寻址目录为当前目录



## 八. 私有registry

使用互联网上的镜像仓库，推送和下载会受带宽影响

目前有两个registry可用，一个是docker提供的`registry`，另一个是CNCF组织提供的`harbor`

### 8.1 docker-registry

#### 8.1.1 容器使用registry

`registry`已经做成镜像，docker-hub上有提供，直接使用`docekr pull registry`拉取镜像，运行即可

配置文件为`/etc/docker/registry/config.yml`

#### 8.1.2 宿主机使用docker-registry

##### 1. 安装

```shell
yum info docekr-registry    # 查看安装包的信息，可查看所属的repo
yum install docker-registry
rpm -ql docker-distribution   # 查看安装生成了哪些文件, docker-registry实际时安装的docker-distribution
```

##### 2. 配置

`docker-distribution`的配置文件为`/etc/docker-distribution/registry/config.yml`

```yaml
version: 0.1
log:
  fields:
    service: registry  # 启动的服务名
storage:
    cache:
        layerinfo: inmemory   # 缓存在内存中
    filesystem:
        rootdirectory: /var/lib/registry   # 数据存放目录
http:
    addr: :5000    # 监听的端口5000
```

##### 3. 启动服务

```shell
systemctl start docker-distribution  # 启动服务
ss -tnl   # 查看端口5000是否监听（服务是否正常启动）
```



**注意**

docker默认基于https通信，若需指定基于http，需在docker配置文件（`/etc/docker/daemon.json`）中修改，之后重启docker服务

```shell
{
	... 
	# 添加
	"insecure-registries":["url1"]
}
```

### 8.2 harbor

vmware-harbor托管在github上的，也做成了容器，部署时需要协同其他组件一起，需要使用docker-compose

## 九. docker 资源限制

默认情况容器没有任何资源限制，默认情况可以使用宿主机的全部资源。docker可以控制容器消耗的CPU、内存、I/O数量（一般只能控制前两个维度，而且需要内核的capability支持）



在linux宿主上，如果内存不够用，将抛出`OOME`(Out Of Memory Exception)，并且开始结束进程以释放内存。

一旦发生OOM，任何进程都可能被杀死，包括docker daemon在内

因此docker特地调整了docker daemon的OOM优先级，避免它被内核杀死，但容器的优先级并未调整



* `lscpu`    查看cpu信息

### 9.1 限制内存

内存从`ram`和`swap`两方面进行限制（https://www.cnblogs.com/wangmo/p/9476490.html），docker run时使用以下参数

* `-m`或`--memory=`    限制容器可用的ram大小，可使用k、b、m、g作为单位

* `--memory-swap`    限制容器可用的swap大小

  * 只有`-m`或`--memory=`有设置时才生效

  | --memory-swap | --memory | 算法                                                         |
  | ------------- | -------- | ------------------------------------------------------------ |
  | 正数S         | 正数M    | 容器可用总空间为S，其中ram为M，swap为(S-M)，若S=M，则无可用swap资源 |
  | 0             | 正数M    | 相当于未设置swap（相当于unset）                              |
  | unset         | 正数M    | 若宿主机启用了swap，则容器可用的swap为 (2*M)                 |
  | -1            | 正数M    | 若宿主机启用了swap，则容器最大可使用所有宿主机的swap资源     |

  **注意**：容器内执行`free`命令看到的swap不具备什么意义

* `--memory-swappiness`    使用swap的倾向性，[0-100]，值越高越常使用swap

* `--memory-reservation`    软内存上线，略小于`--memory`设置的值，用于docker探测低内存时机

* `--kernel-memory`    容器能使用的最大内核内存

* `--oom-kill-disable`    是否oom-kill，也需要设置`--memory`

### 9.2 限制CPU

进程的优先级，数字越小优先级越高

* 实时优先级数字为[0-99]，一般用于内核级的、重要的进程（docker1.13版本后才支持）
* 非实时优先级为[100-139]，一般用于普通进程调度，调度算法一般为CFS（完全公平调度算法）



* `--cpu-shares`     按比例共享分配CPU资源（不需要的让出CPU给需要的使用，需要的按比例分配）
* `--cpus="num"`    容器最多使用几核CPU，可以指定小数`--cpu="1.5"`
* `--cpuset-cpus`    容器只能使用那几个CPU，如`1,3`只能使用第1和第3个核心，`0-3`只能使用第0到第3个核心
* `--cpu-period`    使用`--cpus`代替
* `--cpu-quota`    使用`--cpus`代替
