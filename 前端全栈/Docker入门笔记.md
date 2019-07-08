# Docker入门笔记

工作需要，部署过Sentry，因此接触到docker。本文作为docker的入门学习中的个人笔记，记录docker的入门知识。很多地方只记录了个人认为比较重要的点，具体更详细的可以参考一下文档
**学习资料**：

* [docker入门到实践](https://yeasy.gitbooks.io/docker_practice)
* [docker入门教程](https://github.com/jaywcjlove/docker-tutorial)

## **基本概念**

#### **镜像**

> 操作系统分为内核和用户空间。对于 Linux 而言，内核启动后，会挂载 root 文件系统为其提供用户空间支持。而 Docker 镜像（Image），就相当于是一个 root 文件系统

#### **容器**

> 镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的 类 和 实例 一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等

#### **数据卷**

> 所有的文件写入操作，都应该使用 [数据卷（](https://yeasy.gitbooks.io/docker_practice/data_management/volume.html)[Volume](https://yeasy.gitbooks.io/docker_practice/data_management/volume.html)[）](https://yeasy.gitbooks.io/docker_practice/data_management/volume.html)、或者绑定宿主目录，在这些位置的读写会跳过容器存储层，直接对宿主（或网络存储）发生读写，其性能和稳定性更高

#### **仓库**

> 一个集中的存储、分发镜像的服务

公共仓库：[Docker Hub](https://hub.docker.com/)

加速器： [阿里云加速器](https://cr.console.aliyun.com/#/accelerator)、[DaoCloud](https://www.daocloud.io/mirror#accelerator-doc) [加速器](https://www.daocloud.io/mirror#accelerator-doc) 

国内类似于 Docker Hub 的公开服务： [时速云镜像仓库](https://hub.tenxcloud.com/)、[网易云镜像服务](https://c.163.com/hub#/m/library/)、[DaoCloud](https://hub.daocloud.io/) [镜像市场](https://hub.daocloud.io/)、[阿里云镜像库](https://cr.console.aliyun.com/) 等。 



<br/>

## **操作镜像**

* 获取镜像           `docker pull ubuntu:18.04` 

* 列出顶层镜像   `docker image ls` ,  `docker images` 

* 列出所有镜像    `docker image ls -a`（包含中间层镜像，既其它镜像所依赖的镜像） 

* 列出部分镜像.   `docker image ls ubuntu` 

* 高级用法            `docker image ls --format "{{.ID}}: {{.Repository}}"` 以特定格式显示。 列表包含了 仓库名、标签、镜像 ID、创建时间 以及 所占用的空间 。镜像 ID 则是镜像的唯一标识，一个镜像 可以对应多个 标签 

* 查看镜像、容器、数据卷所占用的空间:   `docker system df` 

* 虚悬镜像**:** 由于新旧镜像同名，旧镜像名称被取消，从而出现仓库名、标签均为 <none> 的镜像除了 docker pull 可能导致这种情况，docker build 也同样可以导致这种现象 

* 虚悬镜像已经失去了存在的价值，是可以随意删除的:   `docker image prune` 

* 删除镜像： ` docker image rm <镜像> `

​      **Untagged** 和 **Deleted:**  镜像的唯一标识是其 ID 和摘要，而一个镜像可以有多个标签, 因此当我们删除了所指定的标签后，可能还有别的标签指向了这个镜像,  Delete 行为就不会发生。所以并非所有的` docker image rm` 都会产生删除镜像的行为，有可能仅仅是取消了某个标签而已. 当该镜像所有的标签都被取消了，该镜像很可能会失去了存在的意义，因此会触发删除行为 



<br/>

## **操作容器**

容器是独立运行的一个或一组应用，以及它们的运行态环境

* 创建一个新的容器并运行一个命令:  `docker run`

  * `-p`:  指定端口映射，格式为：主机(宿主)端口**:**容器端口

    `-d`:  后台运行容器

    `--name="nginx-lb`: 为容器指定一个名称

* 终止容器:              `docker container stop`
* 启动已终止容器:   `docker container start `
* 将一个运行态的容器终止，然后再重新启动它:   `docker container restart`
* 查看容器信息:  `docker container ls`,   `docker ps ` 
  
* 查看所有.   `-a`
  
* 删除容器:      `docker container rm`
* 清理掉所有处于终止状态的容器     `docker container prune`

* 进入容器：   `docker exec -it f6bf8dbe7dcc bash`
  
  * 离开 ：`exit `
* 查看容器信息：`docker inspect f6bf8dbe7dcc`

<br/>

## **数据卷**

默认存放在host机器上的指定目录下`/var/lib/docker/volumes`

* 创建:          `docker volume create my-vol`
* 查看：        `docker volume ls`
* 查看指定：`docker volume inspect my-vol`

* 删除:          `docker volume rm my-vol`
* 清理无主的数据卷：`docker volume prune`
* 数据卷是被设计用来持久化数据的，Docker 不会在容器被删除后自动删除数据卷，如果需要在删除容器的同时移除数据卷。可以在删除容器的时候使用`docker rm -v`
* 挂载主机目录:    使用` --mount ` 标记可以指定挂载一个本地主机的目录到容器中去



<br/>

## **Dockerfile**

### **Dockerfile**定制镜像

Dockerfile 是一个文本文件，其内包含了一条条的 指令(**Instruction**)，每一条指令构建一层镜像，因此每一条指令的内容，就是描述该层应当如何构建

* `FROM`  指定基础镜像 
  * eg： `FROM nginx` 
  * `scratch`表示 这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。FROM scratch, 如果你以 scratch 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在 

* `RUN`  执行命令 
  
   * 伪代码
   
     仅仅使用一个 RUN 指令，并使用 && 将各个所需命令串联起来, 防止建立多层 
   
     Dockerfile 支持 Shell 类的行尾添加 \ 的命令换行方式，以及行首 # 进行注释的格式 
   
     最后还添加了清理工作的命令，删除了为了编译构建所需要的软件，这是因为镜像是多层存储，每一层的东西并不会在下一层被删除，会一直跟随着镜像。因此镜像构建时，一定要确保每一层只添加真正需要添加的东西，任何无关的东西都应该清理掉

```
FROM debian:stretch

RUN buildDeps='gcc libc6-dev make wget' \

​    && mkdir -p /usr/src/redis \

​    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \

​    && make -C /usr/src/redis \

​    && make -C /usr/src/redis install \

​    && rm -rf /var/lib/apt/lists/* \
```



* 使用 `Dockerfile`创建镜像:   `docker build [选项]  <上下文路径/URL/-> `
  * eg： `docker build -t nginx:v3  .`  ,  `-t nginx:v3 `指定了镜像的名称  
  
    注意最后有个点，代表使用当前路径的 Dockerfile 进行构建。如果认为路径是在指定 Dockerfile 所在路径，这么理解其实是不准确的。如果对应上面的命令格式，会发现，这是在指定 上下文路径， 实际上是在指定上下文的目录，docker build 命令会将该目录下的内容打包交给 Docker 引擎以帮助构建镜像，
  
  * 直接用 **Git repo** 进行构建
  
    docker build 还支持从 URL 构建: `docker build https://github.com/xxx/xxx.git#:11.1` 
  
    这行命令指定了构建所需的 Git repo，并且指定默认的 master 分支，构建目录为 /11.1/，然后 Docker 就会自己去 git clone 这个项目、切换到指定分支、并进入到指定目录后开始构建  



* `docker build`的工作原理

  Docker 在运行时分为 Docker 引擎（也就是服务端守护进程）和客户端工具。Docker 的引擎提供了一组 REST API，被称为 [Docker Remote API](https://docs.docker.com/develop/sdk/)，而如 docker 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。因此，虽然表面上我们好像是在本机执行各种 docker 功能，但实际上，一切都是使用的远程调用形式在服务端（Docker 引擎）完成 

  docker build 命令构建镜像，其实并非在本地构建，而是在服务端，也就是 Docker 引擎中构建的。那么在这种客户端/服务端的架构中，如何才能让服务端获得本地文件呢？ 

  这就引入了上下文的概念

  构建的时候，用户会指定构建镜像上下文的路径，`docker build` 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎 以帮助构建镜像 

  如果观察 docker build 输出，我们其实已经看到了这个发送上下文的过程

  ```
  Sending build context to Docker daemon 2.048kB
  ```

  如果目录下有些东西确实不希望构建时传给 Docker 引擎，那么可以用 `.gitignore` 一样的语法写一个 `.dockerignore `



<br/>



## **docker-Compose**

*  首先介绍几个术语

	* 服务 (service)：一个应用容器，实际上可以运行多个相同镜像的实例。 

	* 项目 (project)：由一组关联的应用容器组成的一个完整业务单元。 

		可见，一个项目可以由多个服务（容器）关联而成，Compose 面向项目进行管理

		> 参考：https://blog.csdn.net/vchy_zhao/article/details/70238432#1

* Compose 的使用方式基本上就是下面的三板斧

	* 定义 Dockerfile 

	*  定义 docker-compose.yml 

	* 运行 docker-compose up 
	
*  Compose命令
  
  > 详见：
  >
  > https://www.jianshu.com/p/808385b9e4aa
  >
  > https://blog.csdn.net/vchy_zhao/article/details/70238432#1
  
  * 创建或重新创建服务使用的镜像：`docker-compose build`
  
    * `--no-cache` 不使用缓存
  
  * 在后台启动并运行所有的容器:  `docker-compose up -d`
  
*  `docker-compose build [options] [SERVICE...]`
  构建（重新构建）项目中的服务容器。
服务容器一旦构建后，将会带上一个标记名，例如对于 web 项目中的一个 db 容器，可能是 web_db。
可以随时在项目目录下运行 `docker-compose build` 来重新构建服务
  
  选项包括：
  
  * `--force-rm` 删除构建过程中的临时容器。 
  
  * `--no-cache` 构建镜像过程中不使用 cache（这将加长构建过程）。 
  
  * `—pull` 始终尝试通过 pull 来获取更新版本的镜像。 

* `run` 在指定服务上执行一个命令

  `**docker-compose run ubuntu ping [docker.com](http://docker.com/)**`

  将会启动一个 ubuntu 服务容器，并执行 ping [docker.com](http://docker.com/) 命令 

  选项：

  * `-d` 后台运行容器。 

  * `--name NAME` 为容器指定一个名字。 

  * `—entrypoint` CMD 覆盖默认的容器启动指令。 

  * `-e KEY=VAL` 设置环境变量值，可多次使用选项来设置多个环境变量。 

  * `-u, --user=""` 指定运行容器的用户名或者 uid。 

  * `--no-deps` 不自动启动关联的服务容器。 

  * `—rm` 运行命令后自动删除容器，d 模式下将忽略。 

  * `-p, --publish=[]` 映射容器端口到本地主机。 

  * `--service-ports` 配置服务端口并映射到本地主机。 

  * `-T` 不分配伪 tty，意味着依赖 tty 的指令将无法运行。 

* `up docker-compose up  [options] [SERVICE…]`

  该命令十分强大，它将尝试自动完成包括构建镜像，（重新）创建服务，启动服务，并关联服务相关容器的一系列操作。

  链接的服务都将会被自动启动，除非已经处于运行状态。

  可以说，大部分时候都可以直接通过该命令来启动一个项目

  默认情况，如果服务容器已经存在，`docker-compose up` 将会尝试停止容器，然后重新创建（保持使用 volumes-from 挂载的卷），以保证新启动的服务匹配 `docker-compose.yml` 文件的最新内容。如果用户不希望容器被停止并重新创建，可以使用 `docker-compose up --no-recreate`。这样将只会启动处于停止状态的容器，而忽略已经运行的服务。如果用户只想重新部署某个服务，可以使用 `docker-compose up --no-deps -d <SERVICE_NAME>` 来重新创建服务并后台停止旧服务，启动新服务，并不会影响到其所依赖的服务



<br/>

## docker-compose.yml

注意每个服务都必须通过 image 指令指定镜像或 build 指令（需要 Dockerfile）等来自动构建生成镜像。 

如果使用 build 指令，在 Dockerfile 中设置的选项(例如：CMD, EXPOSE, VOLUME, ENV 等) 将会自动被获取，无需在 docker-compose.yml 中再次设置 

> 如何编写docker-compose.yml文件，参考：https://www.jianshu.com/p/2217cfed29d7

介绍几个配置项：

* image：指定服务的镜像名称或镜像 ID。如果镜像在本地不存在，Compose 将会尝试拉取这个镜像

* build：服务除了可以基于指定的镜像，还可以基于一份 Dockerfile，在使用 up 启动之时执行构建任务，这个构建标签就是 build，它可以指定 Dockerfile 所在文件夹的路径

  > build: ./dir

* depends_on

  ```dockerfile
  version: '2'
  services:
    web:
      build: .
      depends_on:
        - db
        - redis
    redis:
      image: redis
    db:
      image: postgres
  ```

* extra_hosts  添加主机名的标签，就是往/etc/hosts文件中添加一些记录，与Docker client的--add-host类似

  ```dockerfile
  extra_hosts:
   - "somehost:162.242.195.82"
   - "otherhost:50.31.209.229"
  ==>
  162.242.195.82  somehost
  50.31.209.229   otherhost
  ```

* ports  映射端口的标签
* volumes  挂载一个目录或者一个已存在的数据卷容器

* command  