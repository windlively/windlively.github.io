---
title: docker安装mongo
date: 2020-12-15 10:40:51
tags:
  - 后端开发
  - docker
---
- 拉取镜像  
  `docker pull mongo`
- 选择一个合适的文件夹, 创建mongo目录并进入  
  `mkdir mongo && cd mongo`
- 创建配置文件目录  
  `mkdir conf`
- 创建并编辑配置文件  
  `vim conf/mongod.conf`  
  mongod.config为mongo配置文件, 示例:
  ```yml
  storage:
  journal:
    enabled: true
  engine: wiredTiger
  #如下配置仅对 wiredTiger 引擎生效（3.0 以上版本） 
  wiredTiger:
    engineConfig:
    # wiredTiger缓存工作集（working set）数据的内存大小，单位：GB 
    # 此值决定了wiredTiger与mmapv1的内存模型不同，它可以限制mongod对内存的使用量，而mmapv1则不能（依赖于系统级的mmap）。默认情况下，cacheSizeGB 的值为假定当前节点只部署一个mongod实例，此值的大小为物理内存的一半；如果当前节点部署了多个mongod进程，那么需要合理配置此值。如果mongod部署在虚拟容器中（比如，lxc，cgroups，Docker）等，它将不能使用整个系统的物理内存，则需要适当调整此值。默认值为物理内存的一半。 
      cacheSizeGB: 1.5
  systemLog:
    logAppend: true
  net:
    bindIp: 0.0.0.0

  # 是否开启授权
  security:
    authorization: enabled
  ```

- 创建并运行mongo docker容器，路径映射需要根据机器更改  

  ```bash
  docker run --name mongo \
     --privileged \
     -p 27017:27017 \
     -v ~/docker-app/mongo/data:/data/db \
     -v ~/docker-app/mongo/conf:/data/configdb \
     -v ~/docker-app/mongo/logs:/data/log/ \
     -d  \
     mongo  \
     -f /data/configdb/mongod.conf
  ```
  解释:
  >--name mongo  #容器名  
  >--privileged  #给予权限  
  >-p 27017:27017   #端口映射  
  >-v ~/docker-app/mongo/data:/data/db   #数据文件夹映射（主机:容器）  
  >-v ~/docker-app/mongo/conf:/data/configdb   #配置文件路径映射  
  >-v ~/docker-app/mongo/logs:/data/log/    #日志文件夹路径映射  
  >-d   #后台运行  
  >mongo   # 所使用的镜像  
  >-f /data/configdb/mongod.conf  #使用配置文件启动(路径对应的容器路径，非主机路径)  
- 附一个bash脚本(在mongo文件夹下执行)
  ```bash
  # 创建一个mongo docker容器
  mkdir conf
  touch conf/mongod.conf
  conf_file='conf/mongod.conf'
  cat>$conf_file<<EOF
  storage:
    journal:
      enabled: true
    engine: wiredTiger
    wiredTiger:
    engineConfig:
      cacheSizeGB: 1.5
  systemLog:
    logAppend: true
  net:
    bindIp: 0.0.0.0
  security:
    authorization: enabled
  EOF
  docker pull mongo
  base_dir=$PWD
  docker run --name mongo \
    --privileged \
    -p 27017:27017 \
    -v $base_dir/data:/data/db \
    -v $base_dir/conf:/data/configdb \
    -v $base_dir/logs:/data/log/ \
    -d  \
    mongo  \
    -f /data/configdb/mongod.conf
  ```
  