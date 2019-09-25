##  VITESS水平分库DEMO一例 
                                 
### 作者                                 
windtalkerbj                                 
                                 
### 日期                                 
2019-09-19                               
                                 
### 标签                                 
MYSQL , VITESS , 分库分表，架构

### 提纲  
0，Prework:闲聊

1，背景

2. 目标愿景

3.STEP BY STEP

4. 未完待续


### 0,Prework:闲聊
  为什么要写本文？
  
  1 目前baidu/stackoverflow上的分库样例都是基于自带的customer/order/product,千篇一律,最重要的是，
  
  因为没有从头构建自定义的CELL/KEYSPACE/TABLET,这类文章丢失了很多细节
	
  2 VITESS自带DEMO流程: 建立NO SHARDING DEMO->垂直分库->水平分库，步骤、细节超多，容易让人误入歧途；
    
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

A类表对应KEYSPACE(逻辑数据库名）为k_normal,不分区;里面的表是sys_code_tl,500行左右

B类表对应KEYSPACE(逻辑数据库名）为k_multi，分2个SHARD;对应表1：xxxx_itf_imp_headers，账务头表，字段INTERFACE_NAME表示具体账务类别；

对应表2：xxxx_itf_imp_interfaces，账务拆分接口表，属于明细表，也有INTERFACE_NAME字段；

C类表对应KEYSPACE(逻辑数据库名）为k_acct，分4个SHARD;对应表1：xxxx_itf_ar_interface,还款计划表，字段I_INCOME_PERIOD表示账务所属月份，如201907，

表示该还款计划属于2019/07月份；表2：xxxx_dtl_accounts,字段ACCOUNTING_DATE表示账务日期


	
	
### 2,目标愿景

逻辑数据库划分：
![image](https://github.com/windtalkerbj/blog/blob/master/images/SPACE.png)
	
应用能看到3个逻辑数据库，K_normal,k_multi,k_acct,分别存放非sharding数据、hash sharding数据和日期范围数据



### 3,创建CELL/KEYSPACE/MYSQL逻辑集群

#### 启动etcd,创建z_hscs CELL(CELL==数据中心,写这句话时，感觉牛气冲天，XX在手，江山我有)

CELL=z_hscs etcd-up.sh
运行成功后验证etcd目录已建
vitess@XXXX]$ etcdctl --endpoints "http://127.0.0.1:2379" ls -r /

/vitess

/vitess/global

/vitess/z_hscs


#### 启动管理台服务

CELL=z_hscs vtctld-up.sh

### 创建K_normal keyspace和对应MYSQL逻辑集群

#### 创建k_normal mysql逻辑集群，生成3个MYSQLD实例，分别为MASTER,READONLY,READ_SVC

CELL=z_hscs KEYSPACE=k_normal UID_BASE=100 vttablet-up.sh

./lvtctl.sh InitShardMaster -force k_normal/0 z_hscs-100

对应SQL/VShema脚本:

#### 在k_normal库上建表

./lvtctl.sh ApplySchema -sql-file create_hscs_normal.sql  k_normal

#### 登记k_normal sharding信息
./lvtctl.sh ApplyVSchema -vschema_file vschema_hscs_normal.json k_normal


### 创建K_multi keyspace和2个SHARD

./lvtctl.sh CreateKeyspace  -force k_multi
SHARD=-80   CELL=z_hscs KEYSPACE=k_multi UID_BASE=200 vttablet-up.sh

SHARD=80- CELL=z_hscs KEYSPACE=k_multi UID_BASE=300 vttablet-up.sh

注：-80表示数据对应的KEPPACE_ID从（负无穷，80],80-表示数据对应的KEPPACE_ID从(80,正无穷）

即表示数据在2个shard中均分

./lvtctl.sh ApplySchema -sql-file create_hscs_multi.sql k_multi

./lvtctl.sh ApplyVSchema -vschema_file vschema_hscs_multi.json k_multi

vshema_hscs_multi.json是VSHEMA描述文件，VITESS靠其知道数据在多个SHARD中怎么分布：

例如 表hscs_itf_imp_headers按interface_name字段在2个SHARD上按HASH分布：

"tables": {
        
	"hscs_itf_imp_headers": {
	
            "column_vindexes": [
	    
                {
		
                    "column": "interface_name",
		    
                    "name": "unicode_loose_md5"
		    
                }
		
            ],
	    
            "auto_increment": {
	    
                "column": "header_id",
		
                "sequence": "head_seq"
		
            }
	    
        }
	
例如 表yxhscs_itf_contract_validated按APPLY_NUM(贷款合约编号)字段在2个SHARD上按HASH分布：
	
	"yxhscs_itf_contract_validated": {
	
            "column_vindexes": [\
	    
                {
		
                    "column": "apply_num",
		    
                    "name": "unicode_loose_md5"
		    
                }
		
            ],
	    
            "auto_increment": {
	    
                "column": "contract_validated_id",
		
                "sequence": "alix_con_seq"
		
            }
	    
        }
	
        

### 创建K_acct keyspace和4个SHARD

./lvtctl.sh CreateKeyspace  -force k_acct

SHARD=-40   CELL=z_hscs KEYSPACE=k_acct UID_BASE=400 vttablet-up.sh

SHARD=40-80 CELL=z_hscs KEYSPACE=k_acct UID_BASE=500 vttablet-up.sh

SHARD=80-c0 CELL=z_hscs KEYSPACE=k_acct UID_BASE=600 vttablet-up.sh

SHARD=c0-    CELL=z_hscs KEYSPACE=k_acct UID_BASE=700 vttablet-up.sh

注：-40,80,c0,表示数据对应的KETSPACE_ID被4等分在4个SHARD中


./lvtctl.sh InitShardMaster -force k_acct/-40 z_hscs-400

./lvtctl.sh InitShardMaster -force k_acct/40-80 z_hscs-500

./lvtctl.sh InitShardMaster -force k_acct/80-c0 z_hscs-600

./lvtctl.sh InitShardMaster -force k_acct/c0- z_hscs-700


./lvtctl.sh ApplySchema -sql-file create_hscs_acct.sql k_acct

./lvtctl.sh ApplyVSchema -vschema_file vschema_hscs_acct.json k_acct

                
Shard后如图：


![image](https://github.com/windtalkerbj/blog/blob/master/images/Shard.png)

mysql实例如图：


![image](https://github.com/windtalkerbj/blog/blob/master/images/SERVER.png)

注：因资源有限（16C24G，灌数时VTGATE容易OOM），我修改了vttablet-up.sh,将MYSQL逻辑集群容量由3个改成2个，去掉了做READ_SVC的节点


### 启动VTGATE，访问VITESS(GOD说，要有门，VITESS就有了门）

CELL=z_hscs vtgate-up.sh


运行 mysql -h 127.0.0.1 -P 15306访问VITESS，然后用source 执行SQL文件装入DEMO数据，下图是装载完成后的示例：

![image](https://github.com/windtalkerbj/blog/blob/master/images/data.png)




### 4，未完待续(动态SHARDING、复制...)
