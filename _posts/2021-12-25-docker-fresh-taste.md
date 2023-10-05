# Docker尝鲜

## 安装
桌面用户安装Docker Desktop即可

PS：苹果ARM用户也可以使用x86镜像，因为底层是用QEMU来运行Linux的。

## 对象
image 程序包<br>
repository 包仓库<br>
container 运行的容器

## 操作

查看下载的image
> docker images [OPTIONS] [REPOSITORY[:TAG]]<br>
> docker image ls [OPTIONS] [REPOSITORY[:TAG]]<br>
> docker image list [OPTIONS] [REPOSITORY[:TAG]]<br>

搜索Docker Hub上的image
> docker search [OPTIONS] TERM

删除image
> docker image rm [OPTIONS] IMAGE [IMAGE...]

下载image
> docker image pull [OPTIONS] NAME[:TAG|@DIGEST]

新建一个容器
> docker container run [OPTIONS] IMAGE [COMMAND] [ARG...]

参数有：
1. -d detach，与screen类似
2. -p 端口映射，主机(宿主)端口:容器端口
3. -t 为容器重新分配一个伪输入终端，通常与 -i 同时使用
4. --name 指定名称
5. -v 绑定一个卷，卷是一个持久化的对象，可以映射到主机<br>
包含三个field，使用【:】来分割，所有值需要按照正确的顺序。第一个field是volume的名字，并且在宿主机上唯一，对于匿名volume，第一个field通常被省略；第二个field是宿主机上将要被挂载到容器的path或者文件；第三个field可选，比如说ro

查看所有container，-a查看所有
> docker container ls [OPTIONS]<br>
> docker container ps [OPTIONS]<br>
> docker container list [OPTIONS]

与run类似，仅运行命令
> docker container exec [OPTIONS] CONTAINER [COMMAND] [ARG...]

启动、停止、查询日志
> docker container start [OPTIONS] CONTAINER [CONTAINER...]<br>
> docker container stop [OPTIONS] CONTAINER [CONTAINER...]<br>
> docker container logs [OPTIONS] CONTAINER<br>

## Docker Compose
Docker Compose基本上就是用docker-compose.yml来帮助我们自动化部署多个容器，文件的内容基本上就是我们run时对应的参数，与Dockerfile很相似。
> docker-compose --env-file [.env] -f [FILE] up -d