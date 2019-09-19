##  VITESS源码级环境搭建&架构分析 
                                 
### 作者                                 
windtalkerbj                                 
                                 
### 日期                                 
2019-09-19                               
                                 
### 标签                                 
MYSQL , VITESS , 分库分表，架构   


### 1概述
Vitess是一个用于对MySql进行水平扩展的存储平台。经过优化，它可以像在专用硬件上那样有效地运行在云体系。它集MySql 数据库的很多重要特性和NoSQL数据库的可扩展性于一体。本人因为工作需要，基于源码模式编译和部署了基于本地的VITESS运行环境，唯一的感觉就是非常复杂，自成体系。本文权当管中窥豹，有意于MYSQL大数据量分库分表的同行看完官方文档后，再看本文，可以将‘不明觉厉’中的‘不明’去掉。

### 2.准备工作（ROOT安装，简单暴力！）

0）OS:CentOS Linux release 7.3.1611，16C16G

1）安装GO1.12(VITESS最新版需要GO1.11+）

2）解决VITESS安装过程中被墙问题，在本用户下修改.bash_profile,添加如下

export GO111MODULE=on

export GOPROXY=https://goproxy.io

3）在.bash_profile中设置其他需要的变量

export VTDATAROOT=/data1/vtdataroot

export VTROOT=下载源码在GOPATH/src下

git clone https://github.com/vitessio/vitess.git vitess.io/vitess


### 3，编译

不安装ZOOKEEPER,我想用ETCD！

在bootstrap.sh中删除if [ "zk_ver" "zk_ver" install_zookeeper

fi

然后

1）./bootstrap.sh 1>install.log 2>install.err,开始编译

过程中会输出如下：

creating git hooks

installing gRPC 1.16.0

Installing setuptools, pip, wheel...

Installing collected packages: setuptools, pip, wheel

Successfully installed pip-19.2.3 setuptools-41.2.0 wheel-0.33.6

Collecting virtualenv

Successfully installed virtualenv-16.7.5

Collecting mysql-connector-python

Successfully installed mysql-connector-python-8.0.17 protobuf-3.9.1 six-1.12.0

Successfully installed enum34-1.1.6 futures-3.3.0 grpcio-1.16.0 grpcio-tools-1.16.0

installing protoc 3.6.1

installing Zookeeper 3.4.14

BUILD SUCCESSFUL

installing etcd v3.3.10

installing Consul 1.4.0

installing py-mock 1.0.1

Successfully installed mysql-connector-python-8.0.17 protobuf-3.9.1 six-1.12.0

bootstrap finished - run 'source dev.env' or 'source build.env' in your shell before building.


.2)source dev.env，然后运行

[root@XXX vitess]# env | grep VT

VTDATAROOT=/data1/vtdataroot

VTTOP=/opt/mygo/src/vitess.io/vitess

VTROOT=/opt/mygo/src/vitess.io/vitess

VT_MYSQL_ROOT=/usr/sbin

VTPORTSTART=15000

make build

生成的BIN在VTROOT/bin下

至此，需要运行的VITESS已编译完成（万里长征，你走完第一步，恭喜）


### 4，本地DEMO环境搭建（万里长征的剩余80%...）

1）准备工作：

因VITESS不能用ROOT用户启动（VTCTL进程会强制中断），因此在VITESS源码目录下对web/vthook/lib/pkg/examples/dist/config/bin等目录打包，然后新增vitess用户作为DEMO运行环境

groupadd vitess

useradd -d /data1/vitess -g vitess -m vitess

chown -R vitess:vitess vtdataroot

注：vi /etc/hosts


新增 XXX.XXX.XXX.109 XXDB-XX-109(解决NewActionAgent() failed: FullyQualifiedHostname: 
f
ailed to reverse lookup this machine's local IP (fe80::215:5dff:fe06:1b3e%eth0): lookup fe80::215

:5dff:fe06:1b3e%eth0: unrecognized address)


2).profile调整


VITESS 2019/09/10

export VTROOT=VTROOT

export MYSQL_FLAVOR=MySQL56

export VTDATAROOT=/data1/vtdataroot

export TOPO=etcd2

PATH=HOME/.local/bin:VTTOP/bin


3）LOCAL模式启动

[vitess@XXXX local]$ ./101_initial_cluster.sh

输出日志如下：

enter etcd2 env

add /vitess/global

add /vitess/zone1

add zone1 CellInfo

etcd start done...

enter etcd2 env

Starting vtctld...

Access vtctld web UI at http://xx-155-109:15000

Send commands with: vtctlclient -server xxx-xxx-109:15999 ...

enter etcd2 env
Starting MySQL for tablet zone1-0000000100...
Resuming from existing vttablet dir:

/data1/vtdataroot/vt_0000000100
Starting MySQL for tablet zone1-0000000101...
Resuming from existing vttablet dir:

/data1/vtdataroot/vt_0000000101
Starting MySQL for tablet zone1-0000000102...
Resuming from existing vttablet dir:

/data1/vtdataroot/vt_0000000102
Starting vttablet for zone1-0000000100...
Access tablet zone1-0000000100 at http://xxxx-155-109:15100/debug/status
Starting vttablet for zone1-0000000101...
Access tablet zone1-0000000101 at http://xxxx-155-109:15101/debug/status
Starting vttablet for zone1-0000000102...
Access tablet zone1-0000000102 at http://xxxx-155-109:15102/debug/status
W0910 16:48:53.326948 58253 main.go:64] W0910 08:48:53.325915 reparent.go:182] master-elect tablet zone1-0000000100 is not the shard master, proceeding anyway as -force was used
W0910 16:48:53.327197 58253 main.go:64] W0910 08:48:53.326077 reparent.go:188] master-elect tablet zone1-0000000100 is not a master in the shard, proceeding anyway as -force was used
New VSchema object:
{
"tables": {

"corder": {

},
"customer": {

},
"product": {

}
}
}
If this is not what you expected, check the input data (as JSON parsing will skip unexpected fields).
enter etcd2 env
Access vtgate at http://xxxx-155-109:15001/debug/status


### 5,核心脚本梳理：

启动顺序：etcd->vtctld-up.sh->vttablet-up.sh->向vtctld依次发送管理命令->vtgate-up.sh

etcd-up.sh：

    启动etcd，新建/vitess/global和/vitess/${cell}目录，调用vtctl添加CELL信息
    
vtctl -topo_implementation etcd2 -topo_global_server_address cell -server_address "cell）


vtctld-up.sh:


启动vtctld

vttablet-up.sh：


调用mysqlctl启动mysqld_safe进程，初始化DB

（创建DB:_vt 创建表：_vt.local_metadata,_vt.shard_metadata 创建用户：'vt_dba'@'localhost'，'vt_app'@'localhost'，'vt_appdebug'@'localhost'，'vt_allprivs'@'localhost'，'vt_repl'@'%'，'vt_filtered'@'localhost'，'orc_client_user'@'%'）

  --mysqlctl -log_dir $VTDATAROOT/tmp -tablet_uid $uid -mysql_port $mysql_port init -init_db_sql_file config/init_db.sql
  
    调用vttablet 启动vttablet 进程
    
初始化管理命令：


# 将一个副本(replicas)设置为master,同时设置READ-ONLY为OFF

vtctlclient -server localhost:15999 InitShardMaster -force commerce/0 zone1-100

#创建Schema

vtctlclient -server localhost:15999 ApplySchema -sql-file create_commerce_schema.sql commerce

#创建vschema

vtctlclient -server localhost:15999 ApplyVSchema -vschema_file vschema_commerce_initial.json commerce

### 6,VITESS核心进程梳理

etcd（1个）:

etcd

--data-dir /data1/vtdataroot/etcd/

--listen-client-urls http://localhost:2379

--advertise-client-urls http://localhost:2379

监听端口在env.sh: ETCD_SERVER="localhost:2379"


vtctld（1个）:

vtctld

-topo_implementation etcd2

-topo_global_server_address localhost:2379

-topo_global_root /vitess/global

-cell zone1

-web_dir /data1/vitess/web/vtctld

-web_dir2 /data1/vitess/web/vtctld2/app

-workflow_manager_init

-workflow_manager_use_election

-service_map grpc-vtctl

-backup_storage_implementation file

-file_backup_storage_root /data1/vtdataroot/backups

-log_dir /data1/vtdataroot/tmp

-port 15000

-grpc_port 15999

-pid_file /data1/vtdataroot/tmp/vtctld.pid

grpc_port在vtctld-up.sh中指定，port在env.sh中指定（vtctld_web_port=15000）


mysqld_safe（3个）:


mysqld_safe --defaults-file=/data1/vtdataroot/vt_0000000100/my.cnf

mysqld_safe --defaults-file=/data1/vtdataroot/vt_0000000101/my.cnf

mysqld_safe --defaults-file=/data1/vtdataroot/vt_0000000102/my.cnf

mysqld（3个）:

mysqld --defaults-file=/data1/vtdataroot/vt_0000000101/my.cnf

--basedir=/usr

--datadir=/data1/vtdataroot/vt_0000000101/data

--plugin-dir=/usr/lib64/mysql/plugin

--log-error=/data1/vtdataroot/vt_0000000101/error.log

--pid-file=/data1/vtdataroot/vt_0000000101/mysql.pid

--socket=/data1/vtdataroot/vt_0000000101/mysql.sock

--port=17101

端口控制在vttablet-up.sh:mysql_port_base=uid_base]


vttablet（3个）:

vttablet

-topo_implementation etcd2

-topo_global_server_address localhost:2379

-topo_global_root /vitess/global

-log_dir /data1/vtdataroot/tmp

-log_queries_to_file /data1/vtdataroot/tmp/vttablet_0000000100_querylog.txt

-tablet-path zone1-0000000100

-tablet_hostname 

-init_keyspace commerce

-init_shard 0

-init_tablet_type replica

-health_check_interval 5s

-enable_semi_sync

-enable_replication_reporter

-backup_storage_implementation file

-file_backup_storage_root /data1/vtdataroot/backups

-restore_from_backup

-port 15100

-grpc_port 16100

-service_map grpc-queryservice,grpc-tabletmanager,grpc-updatestream

-pid_file /data1/vtdataroot/vt_0000000100/vttablet.pid

-vtctld_addr http://XXX-XXXX-109:15000/

端口控制在vttablet-up.sh:

port_base=$[15000 + $uid_base]

grpc_port_base=$[16000 + $uid_base]

vtgate（1个）:

vtgate

-topo_implementation etcd2

-topo_global_server_address localhost:2379

-topo_global_root /vitess/global

-log_dir /data1/vtdataroot/tmp

-log_queries_to_file /data1/vtdataroot/tmp/vtgate_querylog.txt

-port 15001

-grpc_port 15991

-mysql_server_port 15306

-mysql_server_socket_path /tmp/mysql.sock

-cell zone1

-cells_to_watch zone1

-tablet_types_to_wait MASTER,REPLICA

-gateway_implementation discoverygateway

-service_map grpc-vtgateservice

-pid_file /data1/vtdataroot/tmp/vtgate.pid

-mysql_auth_server_impl none

port/grpc_port/mysql_server_port在vtgate-up.sh中设置


web_port=15001

grpc_port=15991

mysql_server_port=15306

mysql_server_socket_path="/tmp/mysql.sock"

### 7，MYSQL核心参数梳理：

VITESS用到了MYSQL的GTID、半同步等技术，其中对应每个shard的进程组包含vttablet,还有3个MYSQL进程，分别充当MASTER,SLAVE和READ_SERVER服务

binlog_format = statement

innodb_flush_log_at_trx_commit = 2

read-only

slave_net_timeout = 60

slave_load_tmpdir = /data1/vtdataroot/vt_0000000100/tmp(从库复制load data infile XXX时创建）

transaction-isolation = REPEATABLE-READ


半同步相关

plugin-load = rpl_semi_sync_master=semisync_master.so;rpl_semi_sync_slave=semisync_slave.so

rpl_semi_sync_master_timeout = 1000000000000000000（不使用异步复制）

rpl_semi_sync_master_wait_no_slave = 1


GTID相关

gtid_mode = ON

log_bin

log_slave_updates

enforce_gtid_consist

