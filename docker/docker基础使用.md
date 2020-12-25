# 基础使用

## image

`docker image ls`：查看本地已有image。



### pull 

`docker pull ubuntu:14.04`：拉取镜像

镜像加速：

- ` docker pull registry.docker-cn.com/library/ubuntu:16.04`

- 从[阿里云](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)获取加速链接，加入到`/etc/docker/daemon.json`：

  ```
  {
    "registry-mirrors": ["https://xxxx.mirror.aliyuncs.com"]
  }
  ```

  然后让docker重新加载配置：

  ```
  $ sudo systemctl daemon-reload
  $ sudo systemctl restart docker
  ```

  

### commit

在原有image上，修改container之后，运行`docker (container) commit container-NAME`的方式创建**新的image**。

这种方法创建的image不安全，使用者不知道该image是如何创建的。



### run

`docker run image-name` ：从image创建一个container并运行，参数：

- **-i**：interact，交互式
- **-t**：为容器重新分配一个伪输入终端，通常与 -i 同时使用
- **-d**：后台运行容器，并返回容器ID
- **-p:** 端口映射，格式为：主机(宿主)端口:容器端口



## Dockerfile

详细介绍参考该[文档](https://docs.docker.com/engine/reference/builder/)。

### FROM

- `FROM scratch`：制作base image
- `FROM centos`、`FROM ubuntu:14.04`：使用base image，尽量使用官方的，因为安全



### LABEL

类似注释

```
LABEL maintainer="xxx@bupt.edu.cn"
LABEL version="1.0"
LABEL description="this is description"`
```



### RUN

执行命令

```
RUN yum update && yum install -y vim python-dev
```

由于**每一条RUN命令会创建一个layer**，为了避免无用分层，将多条命令合并成一层。



### EXPOSE

暴露端口，例如：`EXPOSE 5000`



### WORKDIR

设定工作目录，类似`cd`

```
WORKDIR /test  # 如果没有会自动创建test目录
WORKDIR demo
RUN pwd		  # /test/demo
```

**用`WORKDIR`，不要用`RUN cd`，尽量使用绝对目录。**



### ADD   and   COPY

ADD除了可以把文件添加到指定目录，还可以解压缩。

```
ADD hello /		# 添加hello到根目录下
ADD test.tar.gz /	# 添加到根目录并解压

WORKDIR /root
ADD hello test/		# /root/test/hello

WORKDIR /root
COPY hello test/
```

添加远程文件或目录使用wget、curl。



### ENV

设定环境变量，增加可维护性。

```
ENV MYSQL_VERSION 5.6
RUN apt-get install -y mysql-server="${MYSQL_VERSION}" \
	&& rm -rf /var/lib/apt/lists/*
```



### RUN  vs  CMD  vs  ENTRYPOINT

- RUN：执行命令并创建新的image layer

- CMD：设置容器启动后默认执行的命令和参数。
  ​	**如果`docker run image-name`指定了其他命令(例如/bin/bash)，会忽略CMD命令。**

  ​	**如果定义了多个CMD，只有最后一个会执行**

- ENTRYPOINT：设置容器启动时运行的命令

  ​	让容器以应用程序或者服务的形式运行。**不会被忽略，一定会执行。**

  ​	写一个shell脚本作为entrypoint。



#### SHELL格式

```
RUN apt-get install -y vim
CMD echo "hello"
ENTRYPOINT echo "hello"
```

#### EXEC格式

```
RUN ["apt-get", "install", "-y", "vim"]
CMD ["/bin/echo", "hello"]
ENTRYPOINT ["/bin/echo", "hello"]
```



**在exec格式中，命令不是在shell环境下执行的**。因此如果有变量，并不会对变量进行替换（$name 替换为 Docker）。

```
FROM centos
ENV name Docker
ENTRYPOINT echo "hello $name"	#hello Docker
```

```
FROM centos
ENV name Docker
ENTRYPOINT ["/bin/echo", "hello $name"]	#hello $name
ENTRYPOINT ["/bin/bash", "-c", "echo hello $name"]	#hello Docker
```



### build

`docker build -t tagName  .`：通过Dockerfile方式构建image，最后的 **.** 表示基于当前目录的Dockerfile。

`docker history ImageId`：查看该image的层次。

`docker build`的时候，如果在某一步出错了，可以通过`docker run -it ${中间状态的层的id} /bin/bash `进入到中间状态的image中，查找问题。



### push到docker hub

要push的image，在build时，格式为`docker build -t wsfy15/xxx .`。

首先`docker login`，然后`docker push wsfy15/xxx:latest`。

也可以自己搭建本地[registry](https://hub.docker.com/_/registry/)，虽然没有界面，但可以利用[API](https://docs.docker.com/registry/spec/api/)查看状态。但需要在`/etc/docker/daemon.json`文件中加入以下内容：`{"insecure-registries": ["x.x.x.x:portNum"]}`，以及在`/lib/systemd/system/docker.service`的service部分加入一行`EnvironmentFile=-/etc/docker/daemon.json`。



## container

- `docker container ls`: 查看运行中的容器，加`-a`查看所有容器，加`-q`只查看id。
- `docker rm container-id`：删除容器
  - `docker rm $(docker container ls -aq)`  ：删除所有container。
  - `docker rm $(docker container ls -f "status=exited" -q)`  删除所有结束的container。
- `docker rmi imageName`：删除image
- `docker exec -it ${containerId} /bin/bash`：进入运行中的容器
- `docker stop ${name/id}`：暂停容器
- `docker start ${name/id}`：启动容器
- `docker inspect ${id}`：查看容器详细信息
- `docker logs ${id}`：查看容器输出



### 资源限制

限制CPU使用率：

```
$ docker run -it --cpu-period=100000 --cpu-quota=20000 busybox /bin/sh
```

`docker run --help`有关于memory、cpu等的参数。



### 与image的关系

- **在image layer之上建立一个可读写的container layer**，每次创建一个container，相当于在原来的image上加了一层。
- 相当于 类(image) 与 实例(container) 的关系
- image负责app的存储与分发，container负责运行app



### 

