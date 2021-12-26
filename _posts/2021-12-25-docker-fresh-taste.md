# Docker尝鲜
## 安装
略
## 对象
image 程序包<br>
repository 包仓库<br>
container 运行的容器
## 操作
> docker images [OPTIONS] [REPOSITORY[:TAG]]<br>
> docker image ls [OPTIONS] [REPOSITORY[:TAG]]<br>
> docker image list [OPTIONS] [REPOSITORY[:TAG]]<br>

查看下载的image
> docker image rm [OPTIONS] IMAGE [IMAGE...]

删除image
> docker image pull [OPTIONS] NAME[:TAG|@DIGEST]

下载image
> docker container run [OPTIONS] IMAGE [COMMAND] [ARG...]

新建一个容器，部分参数有：
1. -d detach，与screen类似
2. -p 端口映射，主机(宿主)端口:容器端口
3. -t 为容器重新分配一个伪输入终端，通常与 -i 同时使用
4. --name 指定名称
5. -v 绑定一个卷，卷是一个持久化的对象，可以映射到主机<br>
包含三个field，使用【:】来分割，所有值需要按照正确的顺序。第一个field是volume的名字，并且在宿主机上唯一，对于匿名volume，第一个field通常被省略；第二个field是宿主机上将要被挂载到容器的path或者文件；第三个field可选，比如说ro
> docker container ls [OPTIONS]<br>
> docker container ps [OPTIONS]<br>
> docker container list [OPTIONS]

查看所有container，-a查看所有
> docker container exec [OPTIONS] CONTAINER [COMMAND] [ARG...]<br>

与run类似，仅运行命令
> docker container start [OPTIONS] CONTAINER [CONTAINER...]<br>
> docker container stop [OPTIONS] CONTAINER [CONTAINER...]<br>
> docker container logs [OPTIONS] CONTAINER<br>

如名称
