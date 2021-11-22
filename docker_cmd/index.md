# docker - 常用指令




# docker 常用指令



## 验证是否安装成功

```shell
$ docker version  # 查看docker版本
# 或者
$ docker info     # 查看docker系统的信息
```



## 启动docker

```shell
$ systemctl start docker
```



## 查看docker 镜像

```shell
$ docker image ls
或者
$ docker images
```



## 拉取镜像

```shell
$ docker pull <image>:<tag>
```



## 删除镜像

```shell
$ docker rmi [image-id]	          # 删除镜像
$ docker rmi $(docker images -q)	# 删除所有镜像
$ docker rmi $(sudo docker images --filter "dangling=true" -q --no-trunc)   # 删除无用镜像
```



##  列出正在运行的容器(containers)

```shell
$ docker ps
```



## 列出所有的容器(包括不运行的容器)

```shell
$ docker ps -a
```



## 容器相关操作

```  shell
docker exec -it                 # 容器ID sh	进入容器
docker stop ‘container’	        # 停止一个正在运行的容器，‘container’可以是容器ID或名称
docker start ‘container’	      # 启动一个已经停止的容器
docker restart ‘container’	    # 重启容器
docker rm ‘container’	          # 删除容器
docker run -i -t -p :80 LAMP /bin/bash	    # 运行容器并做http端口转发
docker exec -it ‘container’ /bin/bash	      # 进入ubuntu类容器的bash
docker exec -it <container> /bin/sh	        # 进入alpine类容器的sh
docker rm docker ps -a -q	                  # 删除所有已经停止的容器
docker kill $(docker ps -a -q)	            # 杀死所有正在运行的容器，$()功能同``
```



## 查看容器日志

- 查看容器日志

```
docker logs <id/container_name>
```

- 实时查看日志输出

```
docker logs -f <id/container_name> (类似 tail -f) (带上时间戳-t）
```



## docker镜像的导出和导入

```shell
$ docker save -o <保存路径> <镜像名称:标签>       # 导出镜像$ docker save -o ./ubuntu18.tar ubuntu:18.04   # 导出镜像$ docker load --input ./ubuntu18.tar           # 导入镜像
```



## 制作自己的镜像

- **docker commit :**从容器创建一个新的镜像。

```shell
$ docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
```

- **Dockerfile**:命令用于使用 Dockerfile 创建镜像。

```shell
$ docker build [OPTIONS] PATH | URL | -例如：$ docker build -t ztc/ubuntu:v1 . 
```



## 从容器中导出文件

```shell
 # 从主机复制到容器 $ docker cp host_path containerID:container_path # 从容器复制到主机 $ docker cp containerID:container_path host_path
```



## 容器启动参数

```shell
$ docker run ：创建一个新的容器并运行一个命令

语法
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
OPTIONS说明：

-a stdin: 指定标准输入输出内容类型，可选 STDIN/STDOUT/STDERR 三项；

-d: 后台运行容器，并返回容器ID；

-i: 以交互模式运行容器，通常与 -t 同时使用；

-P: 随机端口映射，容器内部端口随机映射到主机的端口

-p: 指定端口映射，格式为：主机(宿主)端口:容器端口

-t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；

--name="nginx-lb": 为容器指定一个名称；

--dns 8.8.8.8: 指定容器使用的DNS服务器，默认和宿主一致；

--dns-search example.com: 指定容器DNS搜索域名，默认和宿主一致；

-h "mars": 指定容器的hostname；

-e username="ritchie": 设置环境变量；

--env-file=[]: 从指定文件读入环境变量；

--cpuset="0-2" or --cpuset="0,1,2": 绑定容器到指定CPU运行；

-m :设置容器使用内存最大值；

--net="bridge": 指定容器的网络连接类型，支持 bridge/host/none/container: 四种类型；

--link=[]: 添加链接到另一个容器；

--expose=[]: 开放一个端口或一组端口；

--volume , -v: 绑定一个卷

# 实例
# 使用docker镜像nginx:latest以后台模式启动一个容器,并将容器命名为mynginx。

$ docker run --name mynginx -d nginx:latest

#使用镜像nginx:latest以后台模式启动一个容器,并将容器的80端口映射到主机随机端口。

$ docker run -P -d nginx:latest

#使用镜像 nginx:latest，以后台模式启动一个容器,将容器的 80 端口映射到主机的 80 端口,主机的目录 /data 映射到容器的 /data。

$docker run -p 80:80 -v /data:/data -d nginx:latest
#绑定容器的 8080 端口，并将其映射到本地主机 127.0.0.1 的 80 端口上。

$ docker run -p 127.0.0.1:80:8080/tcp ubuntu bash
#使用镜像nginx:latest以交互模式启动一个容器,在容器内执行/bin/bash命令。

$ @ztc:~$ docker run -it nginx:latest /bin/bash
```



## 镜像仓库操作

**docker login :** 登陆到一个Docker镜像仓库，如果未指定镜像仓库地址，默认为官方仓库 Docker Hub

**docker logout :** 登出一个Docker镜像仓库，如果未指定镜像仓库地址，默认为官方仓库 Docker Hub

```shell
$ docker login -u 用户名 -p 密码$ docker logout
```


