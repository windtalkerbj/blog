##  VITESSˮƽ�ֿ�DEMOһ�� 
                                 
### ����                                 
windtalkerbj                                 
                                 
### ����                                 
2019-09-19                               
                                 
### ��ǩ                                 
MYSQL , VITESS , ˮƽ�ֿ�ֱ��ܹ���װ��   

### ���  
0������
1. Ԥ��
2. Ŀ��Ը����
3-5. STEP BY STEP
7. δ�������

0,����
	ΪʲôҪд���ģ�
	1 Ŀǰbaidu/stackoverflow�ϵķֿ��������ǻ����Դ���customer/order/product,��������,
	����Ҫ���ǣ���Ϊû�д�ͷ�����Զ����CELL/KEYSPACE/TABLET,�������¶�ʧ�˺ܶ�ϸ��
	2 VITESS�Դ�DEMO����: ����NO SHARDING DEMO->��ֱ�ֿ�->ˮƽ�ֿ⣬����ͷ���ܶ࣬�����Ǿ���棬���ϲ��ˮƽ�֣��Ҿʹ����ˮƽ��
	3 ���⻰:����JAVA������Ϊ�䲻�����Ž�ɽ�����ַ�װ(OTTERʵ�ָ���������ȥ),�Ҿ�ϲ����򵥴ֱ��ģ���Ū������ô���£���˵��Ƹ�ߵ��¶�
	
1,Ԥ��
����Ҫ�����ı�sys_code_tl
CREATE TABLE sys_code_tl (
  CODE_ID bigint(20) NOT NULL,
  LANG varchar(10) NOT NULL,
  DESCRIPTION varchar(240) DEFAULT NULL COMMENT '��������',
  OBJECT_VERSION_NUMBER bigint(20) DEFAULT '1',
  REQUEST_ID bigint(20) DEFAULT '-1',
  PROGRAM_ID bigint(20) DEFAULT '-1',
  ...
  ���ݼ�¼500����
��ˮ�ű�(VITESS���AUTO INCREMENT����ķ������ο���������˵��
create table head_seq(id int, next_id bigint, cache bigint, primary key(id)) comment 'vitess_sequence';
create table line_seq(id int, next_id bigint, cache bigint, primary key(id)) comment 'vitess_sequence';

��Ҫ�����ı�
CREATE TABLE XXXX_headers (
  HEADER_ID bigint(20) NOT NULL  COMMENT '��ID���������������������',
  SOURCE_SYSTEM_CODE varchar(30) DEFAULT NULL COMMENT '��Դϵͳ',
  BATCH_NUM varchar(100) DEFAULT NULL COMMENT '���κ�',
  INTERFACE_NAME varchar(240) DEFAULT NULL COMMENT '�ӿڶ�������',
  ...
  �ֶ�39�������ݼ�¼ 10W������
CREATE TABLE XXXXX_lines (
  	LINE_ID bigint(20) NOT NULL ,
	HEADER_ID bigint(20) NOT NULL COMMENT 'ͷ��ID�����ڹ���������������������',
	SOURCE_ITERFACE_ID bigint(20) NOT NULL COMMENT '��ԴID��INTERFACE������ID',
	PROCESS_DATE datetime DEFAULT NULL COMMENT '��̨�������ڣ��������ɽ��㵥����',    
	...
  �ֶν�300�������ݼ�¼ 300W��
XXXX_headers��XXXXX_lines���ֶ�HEADER_ID��һ�Զ��ϵ
	
2,Ŀ��Ը����
��һ��CELL(��������)��������KEYSPACE(�߼����ݿ�),һ����k_normal,�������SHARD�����ݣ���һ����k_multi,�����Ҫ����������
����k_multi�������������������������е�XXXX_headers/XXXXX_lines

3,����CELL/KEYSPACE/MYSQL�߼���Ⱥ
#### ����etcd,����z_hscs CELL(CELL==��������,д��仰ʱ���о�ţ�����죬XX���֣���ɽ����)
CELL=z_hscs "$script_root/etcd-up.sh"
#### ��������̨����
CELL=z_hscs "$script_root/vtctld-up.sh"
#### ����K_normal keyspace�Ͷ�ӦMYSQL�߼���Ⱥ
CELL=z_hscs KEYSPACE=k_normal UID_BASE=100 "$script_root/vttablet-up.sh"
sleep 15
#### ��MYSQL�߼���Ⱥ��IDΪ100��MYSQL��ΪMASTER��ͬʱ����READONLY=0
./lvtctl.sh InitShardMaster -force k_normal/0 z_hscs-100

#### ��k_normal�ϴ�������Ҫshard�ı�
./lvtctl.sh ApplySchema -sql-file create_hscs_normal.sql  k_normal
#### ��VITESS�Ǽǲ���Ҫshard�ı��METAINFO
./lvtctl.sh ApplyVSchema -vschema_file vschema_hscs_normal.json k_normal

#### ����VSHEMA(VITESS���������ļ�,������) vschema
./lvtctl.sh ApplyVSchema -vschema_file vschema_hscs_normal.json k_normal

#### ����VTGATE(GOD˵��Ҫ����,Ȼ���������,����VITESS֮·��ͨ��)
CELL=z_hscs "$script_root/vtgate-up.sh"

#### ����K_multi keyspace
./lvtctl.sh CreateKeyspace  k_multi

4,����K_MULTI��Ӧ��SHARDʵ��(2��MYSQL��Ⱥ��ÿ����Ⱥ����3��MYSQLD,�е�MASTER/SLAVE/READ_ONLY_SVC)
#### ����SHARD����
SHARD=-80 CELL=z_hscs KEYSPACE=k_multi UID_BASE=300 "$script_root/vttablet-up.sh"
SHARD=80- CELL=z_hscs KEYSPACE=k_multi UID_BASE=400 "$script_root/vttablet-up.sh"
sleep 15
./lvtctl.sh InitShardMaster -force k_multi/-80 z_hscs-300
./lvtctl.sh InitShardMaster -force k_multi/80- z_hscs-400

5,��MYSQL�����������ǼǷ�������VITESS��
./lvtctl.sh ApplySchema -sql-file create_hscs_sharded.sql k_multi
#### ��VITESS�Ǽ���Ҫshard�ı��METAINFO
./lvtctl.sh ApplyVSchema -vschema_file vschema_hscs_sharded.json k_multi

6,VSHEMA������(VITESS�Ƹ���߲���֮һ,��Ŀǰ���ճ̶Ȼ�����װ�ƣ��ڴ����İ�...)
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
                

7��δ�����(��̬SHARDING������SHARDING-VINDEX...)
