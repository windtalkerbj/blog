##  VITESS水平分库DEMO一例 
                                 
### 作者                                 
windtalkerbj                                 
                                 
### 日期                                 
2019-09-19                               
                                 
### 标签                                 
MYSQL , VITESS , 水平分库分表，架构

### 提纲  
0，Prework:闲聊

1，背景

2. 目标愿景

3-5. STEP BY STEP（etcd启动->创建KEYSPACE->创建表、分区规则->数据导入）

7. 未完待续


### 0,Prework:闲聊
  为什么要写本文？
  
  1 目前baidu/stackoverflow上的分库样例都是基于自带的customer/order/product,让人生厌,最重要的是，
  
  因为没有从头构建自定义的CELL/KEYSPACE/TABLET,这类文章丢失了很多细节
	
  2 VITESS自带DEMO流程: 建立NO SHARDING DEMO->垂直分库->水平分库，步骤、细节超多，读者很容易陷入细节无法自拔；
    
  本文是精简版，大家喜欢水平分，我就带大家水平分
  
  3 题外话:讨厌JAVA就是因为其不够开门建山，各种封装(OTTER实现根本看不进去)
  
  我就喜欢最简单粗暴的，先用最短时间弄明白怎么回事决定是否下场玩，然后再说怎么玩逼格高的事儿
  
  人生苦短
  
  4 通过阅读本文，可以掌握用VITESS分库（垂直分隔）、按HASH/按日期范围分表（水平分割）
  
  5 最好的分库方式就是满足绝大部分业务需求的分库，本文仅从纯技术角度演示下最常用的分库手段和途径，目的是帮助了解VITESS，
  
  演示的分库方式就见仁见智了
  
  
  
  
### 1，背景：

目前本DEMO模拟一个财务类应用的数据库，其中存在3类表：

A是无需分区的表，比如是配置表或者数据量小，或应用中未和其他业务类表在可写事务中；

B是需要根据合约编号（业务要素）或接口类型（财务数据处理基础，根据不同接口写入不同的目标表中）进行HASH均分的表；

C按账务日期存放的账务明细、还款计划等，需要按账务日期进行分表

在本DEMO中，CELL(数据中心）会定义为z_hscs;

涉及到的表：

A类表：sys_code_tl,500行左右

B类表：hscs_itf_imp_headers


在一个CELL(数据中心)下有两个KEYSPACE(逻辑数据库),一个是k_normal,存放无需SHARD的数据，另一个是k_multi,存放需要分区的数据

其中k_multi有两个分区，均分上文中所有的XXXX_headers/XXXXX_lines
	
	
### 1,目标愿景
不需要分区的表：sys_code_tl

CREATE TABLE sys_code_tl (

  CODE_ID bigint(20) NOT NULL,
  
  LANG varchar(10) NOT NULL,
  
  DESCRIPTION varchar(240) DEFAULT NULL COMMENT '快码描述',
  
  OBJECT_VERSION_NUMBER bigint(20) DEFAULT '1',
  
  REQUEST_ID bigint(20) DEFAULT '-1',
  
  PROGRAM_ID bigint(20) DEFAULT '-1',
  
  ...
  
  数据记录500多条
  
流水号表(VITESS解决AUTO INCREMENT问题的方案，参看官网不多说）

create table head_seq(id int, next_id bigint, cache bigint, primary key(id)) comment 'vitess_sequence';

create table line_seq(id int, next_id bigint, cache bigint, primary key(id)) comment 'vitess_sequence';


需要分区的表：

CREATE TABLE XXXX_headers (

  HEADER_ID bigint(20) NOT NULL  COMMENT '表ID，主键，供其他表做外键',
  
  SOURCE_SYSTEM_CODE varchar(30) DEFAULT NULL COMMENT '来源系统',
  
  BATCH_NUM varchar(100) DEFAULT NULL COMMENT '批次号',
  
  INTERFACE_NAME varchar(240) DEFAULT NULL COMMENT '接口定义名称',
  
  ...
  
  字段39个，数据记录 10W条左右
  
CREATE TABLE XXXXX_lines (

  	LINE_ID bigint(20) NOT NULL ,
	
	HEADER_ID bigint(20) NOT NULL COMMENT '头表ID，用于关联本行数据所属的批次',
	
	SOURCE_ITERFACE_ID bigint(20) NOT NULL COMMENT '来源ID，INTERFACE表主键ID',
	
	PROCESS_DATE datetime DEFAULT NULL COMMENT '后台处理日期，比如生成结算单日期', 
	
	...
	
  字段近300个，数据记录 300W条
  
XXXX_headers和XXXXX_lines按字段HEADER_ID，一对多关系

	


### 3,创建CELL/KEYSPACE/MYSQL逻辑集群

#### 启动etcd,创建z_hscs CELL(CELL==数据中心,写这句话时，感觉牛气冲天，XX在手，江山我有)

CELL=z_hscs "$script_root/etcd-up.sh"

#### 启动管理台服务

CELL=z_hscs "$script_root/vtctld-up.sh"

#### 创建K_normal keyspace和对应MYSQL逻辑集群

CELL=z_hscs KEYSPACE=k_normal UID_BASE=100 "$script_root/vttablet-up.sh"

sleep 15

#### 将MYSQL逻辑集群中ID为100的MYSQL置为MASTER，同时设置READONLY=0
./lvtctl.sh InitShardMaster -force k_normal/0 z_hscs-100

#### 在k_normal上创建不需要shard的表

./lvtctl.sh ApplySchema -sql-file create_hscs_normal.sql  k_normal

#### 在VITESS登记不需要shard的表的METAINFO

./lvtctl.sh ApplyVSchema -vschema_file vschema_hscs_normal.json k_normal


#### 创建VSHEMA(VITESS分区规则文件,后面聊) vschema

./lvtctl.sh ApplyVSchema -vschema_file vschema_hscs_normal.json k_normal


#### 启动VTGATE(GOD说，要有门,然后就有了门,连接VITESS之路已通！)

CELL=z_hscs "$script_root/vtgate-up.sh"


#### 创建K_multi keyspace

./lvtctl.sh CreateKeyspace  k_multi


### 4,创建K_MULTI对应的SHARD实例(2个MYSQL集群，每个集群包含3个MYSQLD,承担MASTER/SLAVE/READ_ONLY_SVC)

#### 两个SHARD启动

SHARD=-80 CELL=z_hscs KEYSPACE=k_multi UID_BASE=300 "$script_root/vttablet-up.sh"

SHARD=80- CELL=z_hscs KEYSPACE=k_multi UID_BASE=400 "$script_root/vttablet-up.sh"

sleep 15

./lvtctl.sh InitShardMaster -force k_multi/-80 z_hscs-300

./lvtctl.sh InitShardMaster -force k_multi/80- z_hscs-400


### 5,在MYSQL创建分区表并登记分区规则到VITESS上

./lvtctl.sh ApplySchema -sql-file create_hscs_sharded.sql k_multi

#### 在VITESS登记需要shard的表的METAINFO

./lvtctl.sh ApplyVSchema -vschema_file vschema_hscs_sharded.json k_multi


### 6,VSHEMA样例：

unshard:

{

    "sharded": false,
    
    "tables": {
    
        "head_seq": {
	
            "type": "sequence"
	    
        },
	
        "line_seq": {
	
            "type": "sequence"
	    
        },
	
        "sys_code_tl":{
	
        }
	
    }
    
}

shard:

{

    "sharded": true,
    
    "vindexes": {
    
        "hash": {
	
            "type": "hash"
	    
        }
	
    },
    
    "tables": {
    
        "hscs_itf_imp_headers": {
	
            "column_vindexes": [
	    
                {
		
                    "column": "header_id",
		    
                    "name": "hash"
		    
                }
		
            ],
	    
            "auto_increment": {
	    
                "column": "header_id",
		
                "sequence": "head_seq"
		
            }
	    
        },
	
        "hscs_itf_imp_lines": {
	
            "column_vindexes": [
	    
                {
		
                    "column": "header_id",
		    
                    "name": "hash"
		    
                }
		
            ],
	    
            "auto_increment": {
	    
                "column": "line_id",
		
                "sequence": "line_seq"
		
            }
	    
        }
	
    }
    
}

                

### 7，未完待续(动态SHARDING、各种SHARDING-VINDEX...)
