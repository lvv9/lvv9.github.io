# Docker尝鲜

## 安装
桌面用户安装Docker Desktop即可

PS：苹果ARM用户也可以使用x86镜像，因为底层是用QEMU来运行Linux的。
PPS：关于容器的UID https://www.cnblogs.com/sparkdev/p/9614164.html
PPPS：Linux的Docker Desktop和Docker Engine实现原理不同，前者需要宿主支持虚拟化，后者可以通过Portainer进行GUI管理。

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
3. -i Keep STDIN open even if not attached
4. -t 为容器重新分配一个伪终端（标准输入、标准输出、标准错误），通常与 -i 同时使用
5. --name 指定名称
6. -v 绑定一个卷，卷是一个持久化的对象，可以映射到主机<br>
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

## build docker image
1. Spring Boot应用开发
2. 准备Dockerfile
```text
FROM openjdk:8-alpine

COPY *.jar /app.jar
EXPOSE 8080
CMD ["java", "-jar", "/app.jar"]
```
FROM 基础镜像
COPY 将指定文件复制到容器中，官方推荐，而ADD支持更多
EXPOSE 声明端口，必须与启动命令参数-p匹配，或-P时随机映射
CMD\ENTRYPOINT 有三种情况：
- CMD ["executable","param1","param2"] (exec form, this is the preferred form)
- CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
- CMD command param1 param2 (shell form)
> There can only be one CMD instruction in a Dockerfile. If you list more than one CMD then only the last CMD will take effect.
> The main purpose of a CMD is to provide defaults for an executing container. These defaults can include an executable, or they can omit the executable, in which case you must specify an ENTRYPOINT instruction as well.
> If the user specifies arguments to docker run then they will override the default specified in CMD.

推荐第二种，前两种直接运行而无shell进程

RUN 构建过程中执行的命令
> The RUN instruction will execute any commands in a new layer on top of the current image and commit the results. The resulting committed image will be used for the next step in the Dockerfile.
> Layering RUN instructions and generating commits conforms to the core concepts of Docker where commits are cheap and containers can be created from any point in an image's history, much like source control.

3. 镜像构建
   将Dockerfile与jar放在一起，执行docker build -t [tag] .
4. 运行
```shell
docker container run -d -p 8080:8080 [tag]
```
5. 发布
   略，另外构建过程可以用maven插件来实现自动化