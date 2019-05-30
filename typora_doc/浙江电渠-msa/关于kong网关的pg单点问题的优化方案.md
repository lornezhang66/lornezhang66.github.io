### 关于kong网关的pg单点问题的优化方案

> 2019年4月23日15:14:02

[TOC]

#### 概述

##### 故障描述

​		2019年3月26日根据云总控反馈， 发现10.70.62/61/46网段批量出现通断，影响业务Kong网关相关联的数据库postgres sql出现宕机，网关在持续运行一段时间后出现宕机故障，导致相关联的宽带中心，统一搜索服务无法使用。



##### 故障分析

> 现网网关应用存在以下缺陷，对于postgres sql有强依赖的关系。

- 网关启动时，应用会读取pgsql数据库apis表的数据加载到cache（内存）。
- 网关正常运行时会每隔5秒连接一下数据库保持心跳状态，并监听数据是否修改，如果修改会将修改的内容加载到cache。
- 网关正常运行时会设置一个cache的有效时间(默认3600s)，缓存失效后会读取数据库,重新将数据刷进缓存。



##### 解决方案

> 解决思路：报障postgres sql的高可用或者解耦kong网关和postgres sql的强依赖关系。

- 部署PG多机房的集群，尽量报障pgsql的高可用。
- 使用网关的DB-less模式，不使用pgsql数据库作为数据源，改为yml格式的配置文件。



#### PG集群方案

> 现网网关的pg数据库为同机房的主备节点，主节点有问题之后要手动切换至备节点，云总控的故障出现在同一个机房，导致主备两个节点的服务器都处于不可用状态。最终导致kong网关宕机。



##### 部署架构

双机房：三墩，石桥

部署两个或者两个以上节点，主从复制

![img](file:///C:\Users\ThinkPad\AppData\Local\Temp\ksohtml4136\wps1.jpg) 

Kong访问数据库时有两个或两个以上节点，通过域名访问，分发到多个节点，从数据库同步主数据库数据。

##### 部署方案

三台电脑：一台主数据库和两台从数据库

- 安装Postgresql9

  ```shell
  tar -zxvf postgresql-9.1.3.tar.gz    #解压
  
  cd postgresql-9.1.3
  
  ./configure --prefix /home/proxy_pg/pgsql  #配置postgresql安装目录
  # 这里需要安装基础的库(gcc、readline、zlib、) #可以不必理会，make时会提示
  make                         #编译
  make install                    #安装
  ```

  

- 添加postgres账户

```shell
useradd postgres                   #添加账户

passwd postgres                    #之后输入密码
#之后进入postgres用户

su postgres
```

- 初始化主数据库

```shell
mkdir data                        #在你想要存放数据的地方创建data文件夹

bin/initdb –D ../data/               #初始化数据库

# 修改data/postgresql.conf

listen_addresses = '0.0.0.0'

port = 5433                        #可以任意更改你想要的

wal_level = hot_standby

max_wal_senders = 30

修改data/pg_hba.conf

host    replication     all             0.0.0.0 0.0.0.0         trust

# 启动主数据库

bin/pg_ctl start -D ../data/

# 检测数据库是否启动成功

./psql -d postgres

psql (9.1.3)

Type "help" for help.

postgres=#                           #说明启动成功了

```

- 基础备份

```shell
# 基本流程：在主数据库服务器上执行pg_start_backup(),复制data目录，在执行pg_stop_backup()。 

./psql –d postgres

postgres=# select pg_start_backup(‘’);

#这姓这个方法后，所有请求在写日志之后不会再刷新到磁盘。直到执行pg_stop_backup()这个函数。

#下面需要拷贝一份data目录，并通过scp复制到子数据库中

cp –R data /data_bac
```



- 创建从数据库(standby)

```shell
#通过scp方式拷贝data_bac目录到从数据库下(当然也可以通过其他方式)

scp -r data_bac/ postgres@192.168.30.199:/home/proxy_pg/pgsql/

#进入从数据库服务器，进入刚刚拷贝过来的data_bac目录下

cd data_bac

#修改postgres.conf

port = 5433 #改成你想的端口

hot_standby = on

#增加recovery.conf配置下连接的主数据库信息(ip、端口、用户)

standby_mode = 'on'

primary_conninfo = 'host=192.168.30.150 port=5433 user=postgres'

#删除pid文件

rm postmaster.pid
```



- 启动从数据库

```shell
 bin/pg_ctl start -D ../data_bac/
```

- 停止主数据库基础备份

```shell
postgres=# select pg_stop_backup();
```



这里要注意的是：8和9步可以颠倒，因为先停止备份后，当从数据库启动会，会自动连接主数据库，拉取服务器自基础备份后的事务日志，然后，对事物日志进行重演。

- 测试

\#为了方便查看数据库，我安装了pgAdminII

当在主数据库中创建一张表并插入三条数据后，观察从数据库：

![img](file:///C:\Users\ThinkPad\AppData\Local\Temp\ksohtml4136\wps2.png) 

```shell
./createdb lengzijian -p 5433      #主从库全部都会创建lengzijian数据库

```





#### DB-less方案

> 现网网关API的存储介质为PG数据库，无论将cache的有效时间改为永久有效还是关掉监听，心跳机制都无法实现与PG数据库的完全解耦，DB-less方案可以将API配置信息的存储介质改为配置文件，最终实现与PG数据库的完全解耦。



##### 部署架构

![](D:\docs\typora_doc\img\未命名文件.png)

##### 部署方案

- 升级Kong到1.11版本。

- 关闭database的配置。

  ```json
  configuration: - {
  ...
  database: "off",
  ...
  }
  ```

- 配置kong.yml配置文件

  ```yml
  _format_version: "1.1"
  
  services:
  - name: my-service
    url: https://example.com
    routes:
    - name: my-route
      paths:
      - /
  
  ```

  

#### 方案对比

> 两种方案各有优劣，DB的数据可维护性要高于配置文件的可维护性。但同时也增加了一个故障点，运维难度要高于配置文件。



| 方案            | 优点                                                         | 缺点                                     |
| --------------- | ------------------------------------------------------------ | ---------------------------------------- |
| **PG集群方案**  | API的可维护性高，业务中心注册服务之后应用会自动拉取新注册的服务注册到Kong网关。 | 高可用方案比较难保障。运维难度相对较高。 |
| **DB-less方案** | 维护成本低，解决了PG耦合的问题。                             | **每次有新接口注册要维护yml配置文件。**  |





