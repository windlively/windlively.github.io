---
title: canal在docker下的搭建(配合canal-admin)
date: 2021-03-07 17:44:54
tags: 
  - 后端开发
  - docker
  - MySQL
  - canal
---
## MySQL配置

- 对于自建 MySQL , 需要先开启 Binlog 写入功能，配置 binlog-format 为 ROW 模式，my.cnf 中配置如下

    ```
    [mysqld]
    log-bin=mysql-bin # 开启 binlog
    binlog-format=ROW # 选择 ROW 模式
    server_id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复

    ```

    - 注意：针对阿里云 RDS for MySQL , 默认打开了 binlog , 并且账号默认具有 binlog dump 权限 , 不需要任何权限或者 binlog 设置,可以直接跳过这一步
- 授权 canal 链接 MySQL 账号具有作为 MySQL slave 的权限, 如果已有账户可直接 grant

    ```sql
    CREATE USER canal IDENTIFIED BY 'canal';
    GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
    -- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
    FLUSH PRIVILEGES;
    ```

    PS：我本地使用的MySQL 8.0，实测不用修改此配置也可正常使用

## canal容器的创建与启动

1. 拉取所需要的镜像

    ```bash
    # canal-admin
    🍓 ➜  ~ docker pull canal/canal-admin
    # canal-server
    🍓 ➜  ~ docker pull canal/canal-server
    ```

2. 下载canal-admin的运行脚本

    ```bash
    wget https://raw.githubusercontent.com/alibaba/canal/master/docker/run_admin.sh
    ```

3. 启动并运行canal-admin

    ```bash
    # 以8089端口启动canal-admin
    sh run_admin.sh -e server.port=8089 \
             -e canal.adminUser=admin \
             -e canal.adminPasswd=admin

    # 指定外部的mysql作为admin的库
    sh run_admin.sh \
        -e server.port=8089 \
        -e spring.datasource.address=host.docker.internal \
        -e spring.datasource.database=xxx \
        -e spring.datasource.username=root \
        -e spring.datasource.password=xxx
    ```

4. 打开浏览器，访问[http://localhost:8089](http://localhost:8089)，默认账号密码为admin/123456，首页如图所示

    ![](https://oscimg.oschina.net/oscnet/up-9fe870724e9ad4581f18e9a4be241c1fcaf.png)

5. 下载canal-server的运行脚本

    ```bash
    wget https://raw.githubusercontent.com/alibaba/canal/master/docker/run.sh
    ```

6. admin管理模式启动并运行canal-server，本案例以单机模式为例，集群模式需手动创建一个集群，并且提供zookeeper地址，然后启动参数加上集群名称即可，配置与单机模式一样，只不过一个集群共用一个server配置

    ```bash
    # 以单机模式启动
    run.sh -e canal.admin.manager=127.0.0.1:8089 \
             -e canal.admin.port=11110 \
             -e canal.admin.user=admin \
             -e canal.admin.passwd=4ACFE3202A5FF5CF467898FC58AAB1D615029441

    # 集群模式启动，自动加入test集群
    # 如果使用集群模式，需要在canal-admin管理页面创建集群，同一个集群使用相同的配置文件
    run.sh -e canal.admin.manager=127.0.0.1:8089 \
             -e canal.admin.port=11110 \
             -e canal.admin.user=admin \
             -e canal.admin.passwd=4ACFE3202A5FF5CF467898FC58AAB1D615029441 
             -e canal.admin.register.cluster=test
    ```

    PS：在docker容器中，若非主机网络模式，127.0.0.1并非为主机的地址，Mac版可使用**host.docker.internal**访问到主机，Linux暂不支持，需要手动在容器中查询到主机的ip，替换掉上面的127.0.0.1

7. 启动成功后，刷新admin页面的server列表，会出现刚刚启动的canal-server：

    ![](https://oscimg.oschina.net/oscnet/up-4aadd69d740610d80bfaf9c01b2047db717.png)

    至此，canal的创建与启动已经完成，在管理页面进行配置操作即可

## canal核心配置介绍

canal的配置分为server配置和instance配置，一个server可包含多个instance，部分配置

### server配置

```properties
# 服务模式，支持tcp, kafka, rocketMQ, rabbitMQ
canal.serverMode = kafka

### kafka配置
# 该值为false时，发送的消息为二进制压缩格式，需要客户端使用protobuf工具解析，为true时，发送json文本
canal.mq.flatMessage = true
# kafka服务器地址
kafka.bootstrap.servers = host.docker.internal:9092
```

### instance配置

```properties
# 监听的数据库地址
canal.instance.master.address=host.docker.internal:3306
# 数据库的用户名
canal.instance.dbUsername=root
# 数据库密码
canal.instance.dbPassword=123456
# 表名过滤，正则表达式，(${库名}.${表名})
canal.instance.filter.regex=.+\\..+

# kafka topic名称，所有的消息都将放入此topic
# canal.mq.topic=example
# 根据库名可表名动态topic
canal.mq.dynamicTopic=.+\\..+
# 发送分区
canal.mq.partition=0
# 分区数量
#canal.mq.partitionsNum=3
# 根据库名和表名计算出发送分区(Hash)，可控制同一个库/表有序
#canal.mq.partitionHash=test.table:id^name,.*\\..*

### 动态topic和partition的详细说明
# canal.mq.dynamicTopic 表达式说明
# canal 1.1.3版本之后, 支持配置格式：schema 或 schema.table，多个配置之间使用逗号或分号分隔

# 例子1：test\\.test 指定匹配的单表，发送到以test_test为名字的topic上
# 例子2：.*\\..* 匹配所有表，则每个表都会发送到各自表名的topic上
# 例子3：test 指定匹配对应的库，一个库的所有表都会发送到库名的topic上
# 例子4：test\\..* 指定匹配的表达式，针对匹配的表会发送到各自表名的topic上
# 例子5：test,test1\\.test1，指定多个表达式，会将test库的表都发送到test的topic上，test1\\.test1的表发送到对应的test1_test1 topic上，其余的表发送到默认的canal.mq.topic值
# 为满足更大的灵活性，允许对匹配条件的规则指定发送的topic名字，配置格式：topicName:schema 或 topicName:schema.table

# 例子1: test:test\\.test 指定匹配的单表，发送到以test为名字的topic上
# 例子2: test:.*\\..* 匹配所有表，因为有指定topic，则每个表都会发送到test的topic下
# 例子3: test:test 指定匹配对应的库，一个库的所有表都会发送到test的topic下
# 例子4：testA:test\\..* 指定匹配的表达式，针对匹配的表会发送到testA的topic下
# 例子5：test0:test,test1:test1\\.test1，指定多个表达式，会将test库的表都发送到test0的topic下，test1\\.test1的表发送到对应的test1的topic下，其余的表发送到默认的canal.mq.topic值
# 大家可以结合自己的业务需求，设置匹配规则，建议MQ开启自动创建topic的能力

# canal.mq.partitionHash 表达式说明
# canal 1.1.3版本之后, 支持配置格式：schema.table:pk1^pk2，多个配置之间使用逗号分隔

# 例子1：test\\.test:pk1^pk2 指定匹配的单表，对应的hash字段为pk1 + pk2
# 例子2：.*\\..*:id 正则匹配，指定所有正则匹配的表对应的hash字段为id
# 例子3：.*\\..*:$pk$ 正则匹配，指定所有正则匹配的表对应的hash字段为表主键(自动查找)
# 例子4: 匹配规则啥都不写，则默认发到0这个partition上
# 例子5：.*\\..* ，不指定pk信息的正则匹配，将所有正则匹配的表,对应的hash字段为表名
# 按表hash: 一张表的所有数据可以发到同一个分区，不同表之间会做散列 (会有热点表分区过大问题)
# 例子6: test\\.test:id,.\\..* , 针对test的表按照id散列,其余的表按照table散列
# 注意：大家可以结合自己的业务需求，设置匹配规则，多条匹配规则之间是按照顺序进行匹配(命中一条规则就返回)
```

### 其余配置可参考官网wiki文档：[alibaba/canal](https://github.com/alibaba/canal/wiki/QuickStart)

## 测试监听

使用kafka客户端测试，由于我使用的动态topic，所以topic名称为：{库名}_{表名}

```bash
🍓 ➜  kafka_2.12-2.2.2 ./bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic test_article
```

手动修改一条该表的数据，即可收到canal发送的数据库变更消息：

![](https://oscimg.oschina.net/oscnet/up-efdcbe72ea74b2357c99b31786999aaf840.png)

## 总结：

总体的搭建过程还是比较简单的，admin模块为我们提供了一个可视化的管理界面，简单易用，单独使用canal-server模块也可以，但是修改配置起来比较麻烦。

此外，在整个过程还是遇到了一些问题的，以下介绍几个踩坑点：

- canal监听时，有些库名和表名是空的，导致在动态topic情况下，kafka提示无效的topic名称，我们在配置中将空库名和表名的消息过滤掉就好，将默认的`.*\\..*`改为`.+\\..+`
- 我的kafka是在本地主机的，需要增加kafka配置`advertised.listeners=PLAINTEXT://host.docker.internal:9092` ，容器中的应用才能正常访问kafka服务端
- 刚开始在启动canal-server时，总是连接admin服务失败，页面上所有接口也无响应，但是docker显示admin容器还在运行，进入canal-admin容器后查看日志无报错，但是后台的java进程已经没有了，困扰了挺久，后来发现我的docker内存配置只有2G，推测是内存不足停掉了，于是将docker内存配置改为8G，问题解决

**本方案仅限于本地学习与测试，生产环境应使用HA模式部署，详见：[https://github.com/alibaba/canal/wiki/AdminGuide#ha模式配置](https://github.com/alibaba/canal/wiki/AdminGuide#ha%E6%A8%A1%E5%BC%8F%E9%85%8D%E7%BD%AE)**

## 附录(GitHub经常无法下载文件，提供一些用到的脚本)

- **GitHub地址：**[https://github.com/alibaba/canal](https://github.com/alibaba/canal)
- **run_admin.sh**

    ```bash
    #!/bin/bash

    function usage() {
        echo "Usage:"
        echo "  run_admin.sh [CONFIG]"
        echo "example :"
        echo "  run_admin.sh -e server.port=8089 \\"
        echo "         -e canal.adminUser=admin \\"
        echo "         -e canal.adminPasswd=admin"
        exit
    }

    function check_port() {
        local port=$1
        local TL=$(which telnet)
        if [ -f $TL ]; then
            data=`echo quit | telnet 127.0.0.1 $port| grep -ic connected`
            echo $data
            return
        fi

        local NC=$(which nc)
        if [ -f $NC ]; then
            data=`nc -z -w 1 127.0.0.1 $port | grep -ic succeeded`
            echo $data
            return
        fi
        echo "0"
        return
    }

    function getMyIp() {
        case "`uname`" in
            Darwin)
             myip=`echo "show State:/Network/Global/IPv4" | scutil | grep PrimaryInterface | awk '{print $3}' | xargs ifconfig | grep inet | grep -v inet6 | awk '{print $2}'`
             ;;
            *)
             myip=`ip route get 1 | awk '{print $NF;exit}'`
             ;;
      esac
      echo $myip
    }

    CONFIG=${@:1}
    #VOLUMNS="-v $DATA:/home/admin/canal-admin/logs"
    PORTLIST="8089"
    PORTS=""
    for PORT in $PORTLIST ; do
        #exist=`check_port $PORT`
        exist="0"
        if [ "$exist" == "0" ]; then
            PORTS="$PORTS -p $PORT:$PORT"
        else
            echo "port $PORT is used , pls check"
            exit 1
        fi
    done

    NET_MODE=""
    case "`uname`" in
        Darwin)
            bin_abs_path=`cd $(dirname $0); pwd`
            ;;
        Linux)
            bin_abs_path=$(readlink -f $(dirname $0))
            NET_MODE="--net=host"
            PORTS=""
            ;;
        *)
            NET_MODE="--net=host"
            PORTS=""
            bin_abs_path=`cd $(dirname $0); pwd`
            ;;
    esac
    BASE=${bin_abs_path}
    DATA="$BASE/data"
    mkdir -p $DATA

    if [ $# -eq 0 ]; then
        usage
    elif [ "$1" == "-h" ] ; then
        usage
    elif [ "$1" == "help" ] ; then
        usage
    fi

    MEMORY="-m 1024m"
    LOCALHOST=`getMyIp`
    cmd="docker run -d -it -h $LOCALHOST $CONFIG --name=canal-admin $VOLUMNS $NET_MODE $PORTS $MEMORY canal/canal-admin"
    echo $cmd
    eval $cmd
    ```

- **run.sh**

    ```bash
    #!/bin/bash

    function usage() {
        echo "Usage:"
        echo "  run.sh [CONFIG]"
        echo "example 1 :"
        echo "  run.sh -e canal.instance.master.address=127.0.0.1:3306 \\"
        echo "         -e canal.instance.dbUsername=canal \\"
        echo "         -e canal.instance.dbPassword=canal \\"
        echo "         -e canal.instance.connectionCharset=UTF-8 \\"
        echo "         -e canal.instance.tsdb.enable=true \\"
        echo "         -e canal.instance.gtidon=false \\"
        echo "         -e canal.instance.filter.regex=.*\\\\\\..* "
        echo "example 2 :"
        echo "  run.sh -e canal.admin.manager=127.0.0.1:8089 \\"
        echo "         -e canal.admin.port=11110 \\"
        echo "         -e canal.admin.user=admin \\"
        echo "         -e canal.admin.passwd=4ACFE3202A5FF5CF467898FC58AAB1D615029441"
        exit
    }

    function check_port() {
        local port=$1
        local TL=$(which telnet)
        if [ -f $TL ]; then
            data=`echo quit | telnet 127.0.0.1 $port| grep -ic connected`
            echo $data
            return
        fi

        local NC=$(which nc)
        if [ -f $NC ]; then
            data=`nc -z -w 1 127.0.0.1 $port | grep -ic succeeded`
            echo $data
            return
        fi
        echo "0"
        return
    }

    function getMyIp() {
        case "`uname`" in
            Darwin)
             myip=`echo "show State:/Network/Global/IPv4" | scutil | grep PrimaryInterface | awk '{print $3}' | xargs ifconfig | grep inet | grep -v inet6 | awk '{print $2}'`
             ;;
            *)
             myip=`ip route get 1 | awk '{print $NF;exit}'`
             ;;
      esac
      echo $myip
    }

    CONFIG=${@:1}
    #VOLUMNS="-v $DATA:/home/admin/canal-server/logs"
    PORTLIST="11110 11111 11112 9100"
    PORTS=""
    for PORT in $PORTLIST ; do
        #exist=`check_port $PORT`
        exist="0"
        if [ "$exist" == "0" ]; then
            PORTS="$PORTS -p $PORT:$PORT"
        else
            echo "port $PORT is used , pls check"
            exit 1
        fi
    done

    NET_MODE=""
    case "`uname`" in
        Darwin)
            bin_abs_path=`cd $(dirname $0); pwd`
            ;;
        Linux)
            bin_abs_path=$(readlink -f $(dirname $0))
            NET_MODE="--net=host"
            PORTS=""
            ;;
        *)
            bin_abs_path=`cd $(dirname $0); pwd`
            NET_MODE="--net=host"
            PORTS=""
            ;;
    esac
    BASE=${bin_abs_path}
    DATA="$BASE/data"
    mkdir -p $DATA

    if [ $# -eq 0 ]; then
        usage
    elif [ "$1" == "-h" ] ; then
        usage
    elif [ "$1" == "help" ] ; then
        usage
    fi

    MEMORY="-m 4096m"
    LOCALHOST=`getMyIp`
    cmd="docker run -d -it -h $LOCALHOST $CONFIG --name=canal-server $VOLUMNS $NET_MODE $PORTS $MEMORY canal/canal-server"
    echo $cmd
    eval $cmd
    ```

- **server配置文件模板(canal.properties)**

    ```properties
    #################################################
    ######### 		common argument		#############
    #################################################
    # tcp bind ip
    canal.ip =
    # register ip to zookeeper
    canal.register.ip =
    canal.port = 11111
    canal.metrics.pull.port = 11112
    # canal instance user/passwd
    # canal.user = canal
    # canal.passwd = E3619321C1A937C46A0D8BD1DAC39F93B27D4458

    # canal admin config
    #canal.admin.manager = 127.0.0.1:8089
    canal.admin.port = 11110
    canal.admin.user = admin
    canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441

    canal.zkServers =
    # flush data to zk
    canal.zookeeper.flush.period = 1000
    canal.withoutNetty = false
    # tcp, kafka, rocketMQ, rabbitMQ
    canal.serverMode = tcp
    # flush meta cursor/parse position to file
    canal.file.data.dir = ${canal.conf.dir}
    canal.file.flush.period = 1000
    ## memory store RingBuffer size, should be Math.pow(2,n)
    canal.instance.memory.buffer.size = 16384
    ## memory store RingBuffer used memory unit size , default 1kb
    canal.instance.memory.buffer.memunit = 1024 
    ## meory store gets mode used MEMSIZE or ITEMSIZE
    canal.instance.memory.batch.mode = MEMSIZE
    canal.instance.memory.rawEntry = true

    ## detecing config
    canal.instance.detecting.enable = false
    #canal.instance.detecting.sql = insert into retl.xdual values(1,now()) on duplicate key update x=now()
    canal.instance.detecting.sql = select 1
    canal.instance.detecting.interval.time = 3
    canal.instance.detecting.retry.threshold = 3
    canal.instance.detecting.heartbeatHaEnable = false

    # support maximum transaction size, more than the size of the transaction will be cut into multiple transactions delivery
    canal.instance.transaction.size =  1024
    # mysql fallback connected to new master should fallback times
    canal.instance.fallbackIntervalInSeconds = 60

    # network config
    canal.instance.network.receiveBufferSize = 16384
    canal.instance.network.sendBufferSize = 16384
    canal.instance.network.soTimeout = 30

    # binlog filter config
    canal.instance.filter.druid.ddl = true
    canal.instance.filter.query.dcl = false
    canal.instance.filter.query.dml = false
    canal.instance.filter.query.ddl = false
    canal.instance.filter.table.error = false
    canal.instance.filter.rows = false
    canal.instance.filter.transaction.entry = false

    # binlog format/image check
    canal.instance.binlog.format = ROW,STATEMENT,MIXED 
    canal.instance.binlog.image = FULL,MINIMAL,NOBLOB

    # binlog ddl isolation
    canal.instance.get.ddl.isolation = false

    # parallel parser config
    canal.instance.parser.parallel = true
    ## concurrent thread number, default 60% available processors, suggest not to exceed Runtime.getRuntime().availableProcessors()
    #canal.instance.parser.parallelThreadSize = 16
    ## disruptor ringbuffer size, must be power of 2
    canal.instance.parser.parallelBufferSize = 256

    # table meta tsdb info
    canal.instance.tsdb.enable = true
    canal.instance.tsdb.dir = ${canal.file.data.dir:../conf}/${canal.instance.destination:}
    canal.instance.tsdb.url = jdbc:h2:${canal.instance.tsdb.dir}/h2;CACHE_SIZE=1000;MODE=MYSQL;
    canal.instance.tsdb.dbUsername = canal
    canal.instance.tsdb.dbPassword = canal
    # dump snapshot interval, default 24 hour
    canal.instance.tsdb.snapshot.interval = 24
    # purge snapshot expire , default 360 hour(15 days)
    canal.instance.tsdb.snapshot.expire = 360

    #################################################
    ######### 		destinations		#############
    #################################################
    canal.destinations = 
    # conf root dir
    canal.conf.dir = ../conf
    # auto scan instance dir add/remove and start/stop instance
    canal.auto.scan = true
    canal.auto.scan.interval = 5

    canal.instance.tsdb.spring.xml = classpath:spring/tsdb/h2-tsdb.xml
    #canal.instance.tsdb.spring.xml = classpath:spring/tsdb/mysql-tsdb.xml

    canal.instance.global.mode = manager
    canal.instance.global.lazy = false
    canal.instance.global.manager.address = ${canal.admin.manager}
    #canal.instance.global.spring.xml = classpath:spring/memory-instance.xml
    canal.instance.global.spring.xml = classpath:spring/file-instance.xml
    #canal.instance.global.spring.xml = classpath:spring/default-instance.xml

    ##################################################
    ######### 	      MQ Properties      #############
    ##################################################
    # aliyun ak/sk , support rds/mq
    canal.aliyun.accessKey =
    canal.aliyun.secretKey =
    canal.aliyun.uid=

    canal.mq.flatMessage = true
    canal.mq.canalBatchSize = 50
    canal.mq.canalGetTimeout = 100
    # Set this value to "cloud", if you want open message trace feature in aliyun.
    canal.mq.accessChannel = local

    canal.mq.database.hash = true
    canal.mq.send.thread.size = 30
    canal.mq.build.thread.size = 8

    ##################################################
    ######### 		     Kafka 		     #############
    ##################################################
    kafka.bootstrap.servers = 127.0.0.1:6667
    kafka.acks = all
    kafka.compression.type = none
    kafka.batch.size = 16384
    kafka.linger.ms = 1
    kafka.max.request.size = 1048576
    kafka.buffer.memory = 33554432
    kafka.max.in.flight.requests.per.connection = 1
    kafka.retries = 0

    kafka.kerberos.enable = false
    kafka.kerberos.krb5.file = "../conf/kerberos/krb5.conf"
    kafka.kerberos.jaas.file = "../conf/kerberos/jaas.conf"

    ##################################################
    ######### 		    RocketMQ	     #############
    ##################################################
    rocketmq.producer.group = test
    rocketmq.enable.message.trace = false
    rocketmq.customized.trace.topic =
    rocketmq.namespace =
    rocketmq.namesrv.addr = 127.0.0.1:9876
    rocketmq.retry.times.when.send.failed = 0
    rocketmq.vip.channel.enabled = false

    ##################################################
    ######### 		    RabbitMQ	     #############
    ##################################################
    rabbitmq.host =
    rabbitmq.virtual.host =
    rabbitmq.exchange =
    rabbitmq.username =
    rabbitmq.password =
    ```

- **instance模板配置(instance.propertios)**

    ```properties
    #################################################
    ## mysql serverId , v1.0.26+ will autoGen
    # canal.instance.mysql.slaveId=0

    # enable gtid use true/false
    canal.instance.gtidon=false

    # position info
    canal.instance.master.address=127.0.0.1:3306
    canal.instance.master.journal.name=
    canal.instance.master.position=
    canal.instance.master.timestamp=
    canal.instance.master.gtid=

    # rds oss binlog
    canal.instance.rds.accesskey=
    canal.instance.rds.secretkey=
    canal.instance.rds.instanceId=

    # table meta tsdb info
    canal.instance.tsdb.enable=true
    #canal.instance.tsdb.url=jdbc:mysql://127.0.0.1:3306/canal_tsdb
    #canal.instance.tsdb.dbUsername=canal
    #canal.instance.tsdb.dbPassword=canal

    #canal.instance.standby.address =
    #canal.instance.standby.journal.name =
    #canal.instance.standby.position =
    #canal.instance.standby.timestamp =
    #canal.instance.standby.gtid=

    # username/password
    canal.instance.dbUsername=canal
    canal.instance.dbPassword=canal
    canal.instance.connectionCharset = UTF-8
    # enable druid Decrypt database password
    canal.instance.enableDruid=false
    #canal.instance.pwdPublicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALK4BUxdDltRRE5/zXpVEVPUgunvscYFtEip3pmLlhrWpacX7y7GCMo2/JM6LeHmiiNdH1FWgGCpUfircSwlWKUCAwEAAQ==

    # table regex
    canal.instance.filter.regex=.*\\..*
    # table black regex
    canal.instance.filter.black.regex=
    # table field filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
    #canal.instance.filter.field=test1.t_product:id/subject/keywords,test2.t_company:id/name/contact/ch
    # table field black filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
    #canal.instance.filter.black.field=test1.t_product:subject/product_image,test2.t_company:id/name/contact/ch

    # mq config
    canal.mq.topic=example
    # dynamic topic route by schema or table regex
    #canal.mq.dynamicTopic=mytest1.user,mytest2\\..*,.*\\..*
    canal.mq.partition=0
    # hash partition config
    #canal.mq.partitionsNum=3
    #canal.mq.partitionHash=test.table:id^name,.*\\..*
    #################################################
    ```
