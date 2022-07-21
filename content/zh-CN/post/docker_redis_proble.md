

+++
title = "redis docker-entrypoint.sh executable file not found"
date = "2022-04-26"
description = "redis docker-entrypoint.sh: executable file not found in $PATH: unknown"
tags = [
    "docker问题"
]
+++


# 使用docker启动redis服务遇到的一个坑

## 问题
最近一个项目需要依赖redis，把之前使用的redis的 docker-compose下载下来，启动一下就OK了。但是因为一个报错，花了1个多小时才解决，在此记录一下。

docker-compose文件如下：
```yaml
version: "3"
services:
  some_redis:
    container_name: some_redis
    image: redis:6.0.16
    volumes:
      - ./conf:/usr/local/
    ports:
      - "6379:6379"
    restart: always
    command: redis-server /usr/local/redis.conf
```
docker-compose up -d 启动后报一下错误：
```
Creating some_redis ... error
ERROR: for some_redis  Cannot start service some_redis: OCI runtime create failed: container_linux.go:380: starting container process caused: exec: "docker-entrypoint.sh": executable file not found in $PATH: unknown
ERROR: for some_redis  Cannot start service some_redis: OCI runtime create failed: container_linux.go:380: starting container process caused: exec: "docker-entrypoint.sh": executable file not found in $PATH: unknown
ERROR: Encountered errors while bringing up the project
```

## 过程
按照提示 docker-entrypoint.sh可执行文件找不到，想着之前启动没遇到这个问题哇，就又尝试重启了好几次，还是报一样的错。
换了镜像版本、增加了PATH环境变量也不行.

上网查了一下这个错误：
大部分的信息是修改redis 中 Dockerfile 命令，增加 下面命令：
`RUN chdmo 777 /usr/local/bin/docker-entrypoint.sh && link -s /usr/local/bin/docker-entrypoint.sh`

或是说因为基础镜像的原因，需要把docker-entrypoint.sh的 #!/bin/bash改为 #/bin/sh，不过这个问题在docker-library/redis 已经修复了

下载了https://github.com/docker-library/redis/blob/master/6.2/Dockerfile redis dockerfile，增加了上面的命令，重新build新的redis镜像，重新执行 `docker-compose up -d ` 还是一样的报错。。。

把dockerfile中 `ENTRYPOINT ["docker-entrypoint.sh"]` 改为绝对路径 `ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]` 也不行。

why？ 

一般情况 遇到这种明显报错问题，1小时还未解决的，我会把问题放一放，一般这种情况要不是非常基础的错误，要不是排查方向错了

晚会儿后我发现，直接执行dockerhub 给的启动的命令是可以的：
```console
$ docker run -v /myredis/conf:/usr/local/etc/redis --name myredis redis redis-server /usr/local/etc/redis/redis.conf
```

## 解决
最后发现是我docker-compose中 redis.conf 文件挂载到`/usr/local` 目录下导致的，挂载目录改为 `/usr/local/etc/redis`下问题解决。

推测应该是容器内redis-server进程没有权限读取`/usr/local/redis.conf`文件导致的 这个错误：

ERROR: for some_redis  Cannot start service some_redis: OCI runtime create failed: container_linux.go:380: starting container process caused: exec: "docker-entrypoint.sh": executable file not found in $PATH: unknown

这个报错信息太不准确了....

后上网搜索了一番，一些docker命令参数顺序不一致也会报这个错

最终的docker-compose.yaml 如下：
```
version: "3"
services:
  some_redis:
    container_name: some_redis
    image: redis:6.2
    volumes:
      - /data/soft/redis/conf:/usr/local/etc/redis
      - /data1/service_data/redis/data:/data
    ports:
      - 6379:6379
    restart: always
    command: redis-server /usr/local/etc/redis/redis.conf

```

---

`https://github.com/huarong8/new_project_docker_compose`

这个项目是我维护的常见的web项目的docker-compose，现在支持redis、mysql、nginx后面会慢慢维护起来

