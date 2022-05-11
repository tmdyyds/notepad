## docker常用命令
```
//其他命令地址
https://www.runoob.com/docker/docker-command-manual.html
```

#### 容器
- run
```
//docker以ubuntu:15.10镜像创建一个容器，然后在容器中执行 bin/echo "Hello world"命令，然后输出结果
docker run ubuntu:15.10 /bin/echo "Hello world"

//交互式容器
docker run -i -t ubuntu:15.10 /bin/bash

//后台模式启动一个容器
docker run -d -name ubuntuwangbin ubuntu:15.10 /bin/sh -c "while true; do echo hello world; sleep 1; done"
```
   * 参数
> -i 允许你对容器内的标准输入 (STDIN) 进行交互。 
> 
> -t 在新容器内指定一个伪终端或终端。
> 
> -d 以后台模式启动一个容器。
> 
> -name 给启动的容器命名
> 
> -P将容器内部使用的网络端口随机映射到我们使用的主机上
> 
> -p 映射固定端口

- stop/restart/exit/exec/attach
``` 
//停止容器
docker stop asdjfk

//退出容器
exit

//重启容器
docker restart adfkjid

//进入容器
docker exec/attach(通过attach进入容器，用exit退出后，容器会停止) dasfk
```

- export/import
```
//导出容器到本地
docker export 1e560fca3906 > ubuntu.tar

//将快照文件 ubuntu.tar 导入到镜像 test/ubuntu:v1
cat docker/ubuntu.tar | docker import - test/ubuntu:v1

//通过url导入
docker import http://example.com/exampleimage.tgz example/imagerepo
```

- logs
```
//查看容器内部的标准输出
docker logs -f sdkafid
```
   * 参数
> -f 让 docker logs 像使用 tail -f 一样来输出容器内部的标准输出

- inspect
```
//docker inspect 来查看 Docker 的底层信息。它会返回一个 JSON 文件记录着 Docker 容器的配置和状态信息
docker inspect wizardly_chandrasekhar
```

- rm
```
//删除容器
docker rm -f 1e560fca3906

//清理掉所有处于终止状态的容器
docker container prune
```

- ps
```
//查看运行中容器
docker ps

//查询最后一次创建的容器
docker ps -l 

```
   * 参数
  > -a 查看所有容器
  > -l 查询最后一次创建的容器

***docker ps 返回值解释***
> CONTAINER ID: 容器 ID。
> 
> IMAGE: 使用的镜像。
> 
> COMMAND: 启动容器时运行的命令。
> 
> CREATED: 容器的创建时间。
> 
> STATUS: 容器状态。
> 
> PORTS: 容器的端口信息和使用的连接类型（tcp\udp）。
> 
> NAMES: 自动分配的容器名称。
> 
> **STATUS有7种值**：
> > created（已创建）
> >
> > restarting（重启中）
> > 
> > running 或 Up（运行中）
> >
> > removing（迁移中）
> >
> > paused（暂停）
> > 
> > exited（停止）
> > 
> > dead（死亡）

#### 镜像
- pull/search
```
//获取镜像 ubuntu镜像
docker pull ubuntu

//查找镜像
docker search ubuntu
```

- commit
```
//镜像拉取运行后，进入容器内 执行apt-get update 命令进行更新
//提交容器副本
dcoker commit -m="更新了内容" -a="" asdjfId runoob/ubuntu:v2
```
   * 参数
> -m 提交的描述信息
> 
> -a 指定镜像作者
> 
> asdjfId 容器ID
> 
> runoob/ubuntu:v2: 指定要创建的目标镜像名(更新后，新的镜像)

- build
```
//通过dockerfile构建镜像
docker build -t runoob/centos:6.7 .
```
   * 参数
> -t 指定要创建的目标镜像名
> 
> .     Dockerfile 文件所在目录，可以指定Dockerfile 的绝对路径

- tag
```
//为镜像添加一个新的标签
docker tag 860c279d2fec runoob/centos:dev(用户名称、镜像源名(repository name)和新的标签名(tag))
```

#### 仓库
```
//官网地址
https://hub.docker.com 
```

**TIPS**
```
//返回所有docker命令
docker

//查看某命令使用教程
docker command(例如:ps) --help
```
