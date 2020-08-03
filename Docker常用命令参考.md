---
title: "Docker常用命令参考"
date: "2019-06-11"
tags: ["Docker"]
categories: ["学习笔记", "Docker"]
---

主要用于记录当前使用过的docker常用命令以及注意事项

## 容器操作

>  ps 查看容器

```bash
runoob@runoob:~$ docker ps
CONTAINER ID   IMAGE          COMMAND                ...  PORTS                    NAMES
09b93464c2f7   nginx:latest   "nginx -g 'daemon off" ...  80/tcp, 443/tcp          myrunoob
96f7f14e99ab   mysql:5.6      "docker-entrypoint.sh" ...  0.0.0.0:3306->3306/tcp   mymysql
```

``` bash
docker ps -a #可以查看全部容器，通常用于查看未运行的容器
```


上面的结果借用于runoob，CONTAINER ID是容器唯一ID，IMAGE是创建该容器使用的镜像，COMMAND是该创建该容器时指定的命令符，PORTS是该容器的端口映射信息，NAMES是容器的名字

> start/stop/restart 启动/停止/重启

``` bash
docker start CONTAINER ID/NAMES #容器的ID和name都可以用作参数来启动或停止
```


>  rm 删除

``` bash
docker rm CONTAINER ID/NAMES
```

>  run 创建一个容器

- **-d:** 后台运行容器，并返回容器ID；
- **-i:** 以交互模式运行容器，通常与 -t 同时使用；
- **-P:** 随机端口映射，容器内部端口**随机**映射到主机的高端口
- **-p:** 指定端口映射，格式为：**主机(宿主)端口:容器端口**
- **-t:** 为容器重新分配一个伪输入终端，通常与 -i 同时使用；
- **--name="nginx-lb":** 为容器指定一个名称；
- **--dns 8.8.8.8:** 指定容器使用的DNS服务器，默认和宿主一致；
- **--dns-search example.com:** 指定容器DNS搜索域名，默认和宿主一致；
- **-h "mars":** 指定容器的hostname；
- **-e username="ritchie":** 设置环境变量；
- **-v localdir:dockerdir: **设置宿主机与docker共享目录，

``` bash
docker run --name mydocker -v /home/linvon/share:/home/docker/share -it "/bin/bash" myimage #使用myimage镜像，采用交互式创建一个名为mydocker的容器
```

> exec 在容器内执行命令

- **-d :**分离模式: 在后台运行
- **-i :**即使没有附加也保持STDIN 打开
- **-t :**分配一个伪终端

``` bash
docker exec -it mydocker /bin/sh /tmp/exec.sh # 可以将需要执行的命令写入脚本放在docker里，直接从宿主机执行命令让docker运行内部的脚本，也可以不加脚本参数直接运行终端，但和直接运行docker的区别不大
```

> attach 连接到正在运行的容器

``` bash
docker start mydocker;docker attach mydocker  # 一般连用两条命令，直接进入容器
```

> export 导出容器tar包

``` bash
docker export -o res.tar mydocker
docker export mydocker > res.tar
```

> commit 备份容器为镜像

``` bash
docker commit mydocker myimage:v1 # 将mydocker容器备份为名为myimage的镜像，并标记TAG为v1
```

> cp 容器与宿主机之间进行文件拷贝

``` bash
docker cp mydocker:/home/a.txt /home/a.txt # 从容器拷贝到宿主机
docker cp /home/a.txt mydocker:/home/a.txt # 从宿主机拷贝到容器
# 需要注意的是，在docker版本小于1.8.0时，不支持从宿主机向容器内拷贝文件，会报错，只能通过共享目录来解决
```



## 镜像操作

>  images 查看镜像

``` bash
runoob@runoob:~$ docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
mymysql                 v1                  37af1236adef        5 minutes ago       329 MB
runoob/ubuntu           v4                  1c06aa18edee        2 days ago          142.1 MB
<none>                  <none>              5c6e1090e771        2 days ago          165.9 MB
httpd                   latest              ed38aaffef30        11 days ago         195.1 MB
```

> rmi 删除镜像

``` bash
docker rmi myimage
```

> save 将镜像导出为tar包

``` bash
docker export -o res.tar myimage
docker export myimage > res.tar
```

> load 加载使用save命令导出的镜像tar

``` bash
docker load -i res.tar
docker load < res.tar
```

> import 加载使用export命令导出的容器tar

``` bash
docker import res.tar myimage:v1
```

