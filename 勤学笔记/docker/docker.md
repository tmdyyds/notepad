## docker常用命令

#### 容器
- run
```
//docker以ubuntu:15.10镜像创建一个容器，然后在容器中执行 bin/echo "Hello world"命令，然后输出结果
docker run ubuntu:15.10 /bin/echo "Hello world"

//交互式容器
docker run -i -t ubuntu:15.10 /bin/bash

//后台模式启动一个容器
docker run -d ubuntu:15.10 /bin/sh -c "while true; do echo hello world; sleep 1; done"
```
> -i 允许你对容器内的标准输入 (STDIN) 进行交互。 -t 在新容器内指定一个伪终端或终端。-d 以后台模式启动一个容器。

- exit
> 退出容器
- ps
```
//查看运行中容器
docker ps
```
docker ps 返回值解释
> CONTAINER ID: 容器 ID。IMAGE: 使用的镜像。COMMAND: 启动容器时运行的命令。CREATED: 容器的创建时间。STATUS: 容器状态。PORTS: 容器的端口信息和使用的连接类型（tcp\udp）。NAMES: 自动分配的容器名称。
> 
> **STATUS有7种值**：created（已创建）restarting（重启中）running 或 Up（运行中）removing（迁移中）paused（暂停）exited（停止）dead（死亡）

#### 镜像
#### 仓库
