---
title: docker安装zookeeper集群(附bash脚本)
date: 2020-12-14 18:27:37
tags:
---
- 准备一个文件夹并进入此文件夹， 例如

  `mkdir -p ~/docker-app/zookeeper && cd ~/docker-app/zookeeper`
  
- 准备三个文件夹并写入zookeeper的myid

  `mkdir -p zoo1/data && echo 1 > zoo1/data/myid`  
  `mkdir -p zoo2/data && echo 2 > zoo2/data/myid`  
  `mkdir -p zoo3/data && echo 3 > zoo3/data/myid`  

- 创建一个docker-compose.yml文件并编辑  

  `vim docker-compose.yml`

  文件内容如下:  

  ```yaml
  version: '3.7'
  networks:
    zk_cluster:
      name: zk_cluster #为集群创建一个网络
      driver: bridge
  #集群中的服务(3个节点)
  services:
    zoo1:
      image: zookeeper #所使用的镜像
      restart: always #容器异常时是否重启
      container_name: zoo1 #容器名称
      ports: #与宿主机的端口映射
        - 2181:2181
        - 8001:8080
      # zookeeper的配置
      environment:
        ZOO_MY_ID: 1
        ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181
      #容器目录映射，此处使用了相对路径
      volumes:
        - ./zoo1/data:/data
        - ./zoo1/datalog:/datalog
      #所使用的网络
      networks:
        - zk_cluster
    zoo2:
      image: zookeeper
      restart: always
      container_name: zoo2
      ports:
        - 2182:2181
        - 8002:8080
      environment:
        ZOO_MY_ID: 2
        ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zoo3:2888:3888;2181
      volumes:
        - ./zoo2/data:/data
        - ./zoo2/datalog:/datalog
      networks:
        - zk_cluster
    zoo3:
      image: zookeeper
      restart: always
      container_name: zoo3
      ports:
        - 2183:2181
        - 8003:8080
      environment:
        ZOO_MY_ID: 3
        ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
      volumes:
        - ./zoo3/data:/data
        - ./zoo3/datalog:/datalog
      networks:
        - zk_cluster
  ```
- 使用docker-compose命令运行  
  
  `docker-compose --project-directory $PWD up -d`

- 附：一键创建docker zookeeper集群并执行的脚本（该方案仅供参考和学习，生产环境需仔细斟酌，合理修改配置参数）
  ```bash
  #! /bin/bash

  mkdir -p zookeeper && cd zookeeper
  DIR=$PWD
  echo "cd $DIR"
  echo "create docker compose config file"
  CONF_YML='docker-compose.yml'
  touch $CONF_YML
  echo "version: '3.7'" > $CONF_YML
  NET='zk_cluster'
  echo 'networks:' >> $CONF_YML
  echo "  $NET:" >> $CONF_YML
  echo "    name: $NET" >> $CONF_YML
  echo "    driver: bridge" >> $CONF_YML
  #节点个数，可以修改此值创建任意数量的容器节点
  let CLUSTER_COUNT=3 
  IMAGE=zookeeper
  echo "services:" >> $CONF_YML
  for i in `seq 1 $CLUSTER_COUNT`
  do
      echo "  zoo$i:" >> $CONF_YML
      echo "    image: $IMAGE" >> $CONF_YML
      echo "    restart: always" >> $CONF_YML
      echo "    container_name: zoo$i" >> $CONF_YML
      echo "    ports:" >> $CONF_YML
      echo "      - $[2180 + $i]:2181" >> $CONF_YML
      echo "      - $[8000 + $i]:8080" >> $CONF_YML
      echo "    environment:" >> $CONF_YML
      echo "      ZOO_MY_ID: $i" >> $CONF_YML
      zk_srv_conf=''
      for j in `seq 1 $CLUSTER_COUNT`
      do
          if test $i != $j;
          then
              zk_srv_conf+="server.$j=zoo$j:2888:3888;2181"
          else
              zk_srv_conf+="server.$j=0.0.0.0:2888:3888;2181"
          fi
          if test $j != $CLUSTER_COUNT;
          then
              zk_srv_conf+=' ';
          fi
      done
      echo "      ZOO_SERVERS: $zk_srv_conf" >> $CONF_YML
      echo "    volumes:" >> $CONF_YML
      echo "      - ./zoo$i/data:/data" >> $CONF_YML
      echo "      - ./zoo$i/datalog:/datalog" >> $CONF_YML
      echo "    networks:" >> $CONF_YML
      echo "      - $NET" >> $CONF_YML
      mkdir -p zoo$i/data
      echo $i > zoo$i/data/myid
  done
  # docker pull zookeeper
  docker-compose --project-directory $DIR up -d
  ```
  