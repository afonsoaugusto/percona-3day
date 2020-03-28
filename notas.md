# O Desafio

## Observações do desafio

* Sistema de Armazenamento foi ampliado em 50% em ambos os servidores de banco de dados
* A replicação está com um atraso (lagging) contínuo e crescente.
* Algumas queries estão retornando resultados diferentes na réplica se comparados aos resultados obtidos pelo master
* A carga de trabalho principal pode ser atribuída à 5 queries ocupando o topo do ranking e uma análise preliminar indica que existe espaço para melhorias em todas elas.
* De tempos em tempos a *replica* se encontra saturada de tal maneira que não é possível fazer uma nova conexão.

### Expectativa

* Identificar os problemas de performance em ambas as máquinas
* Corrijir os problemas de performance em ambas as máquinas
* Solucionar a questão da inconsistência de dados
* Seja mantida a compatibilidade com o workload atual.

### Considerações

Apesar de ter sido sugerida a substituição dos servidores de banco de dados utilizados nesse ambiente por máquinas com uma quantidade maior de recursos, ou ainda a substituição da arquitetura atual, foi confirmado que o duo master-replica atuando em produção é suficiente para suprir as demandas deste ambiente por mais algum tempo considerando que sejam feitos os ajustes necessários para tirar o melhor proveito dos mesmos.

Apesar de gozar de carta branca, atente-se para o fato de que ambos os servidores de banco de dados continuarão recebendo requisições: faço um planejamento adequado de suas ações de modo a minimizar quaisquer impactos que afetem à continuidade das operações, porém não deixe de efetuar as mudanças necessárias.

Escreva um relatório em português apontando todas as ações realizadas, justificando-as e incluindo evidências de que as mesmas contribuíram para solucionar cada um dos pontos ressaltados acima e/ou acarretaram em melhorias na performance do servidor. A avaliação final levará em conta a resolução dos problemas identificados logo a comprovação dos mesmos com evidências comparando o "antes" e o "depois" são imprescindíveis.

Pontos extras para a *correta definição da inconsistência de dados e descrição detalhada do trabalho de otimização das queries*.

### Acesso

O endereço IP que dá acesso ao ambiente é o 157.245.210.47 sendo observado o seguinte esquema de redirecionamento de portas:

* 2022 : master (SSH)
* 2023 : replica (SSH)
* 8080 : PMM (HTTP)

Você encontrará a chave RSA de acesso anexa à este e-mail. Utilize o usuário vagrant para conexões SSH, a frase-chave é:

*Percona DBA Challenge, baby!*

# Compare Nodes OS

http://157.245.210.47:8080/graph/d/node-instance-compare/nodes-compare?orgId=1&refresh=1m&var-interval=$__auto_interval_interval&var-node_name=master&var-node_name=replica&var-service_name=All&var-environment=All&var-cluster=All&var-replication_set=All

* Compare MySQL Instance

http://157.245.210.47:8080/graph/d/mysql-instance-compare/mysql-instances-compare?orgId=1&refresh=1m&var-interval=$__auto_interval_interval&var-region=&var-node_name=All&var-environment=&var-cluster=&var-service_name=master-mysql&var-service_name=replica-mysql&var-node_id=&var-agent_id=&var-service_id=%2Fservice_id%2Fbc27ba2f-8cd8-43fa-a999-bde8369471ba&var-az=&var-node_type=&var-node_model=&var-replication_set=&var-database=All

## Replica

```sql
mysql> show variables like "max_connections";
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 151   |
+-----------------+-------+
1 row in set (0.01 sec)

mysql>

set global max_connections = 300;

mysql> show variables like "%conn%";
+-----------------------------------------------+-----------------+
| Variable_name                                 | Value           |
+-----------------------------------------------+-----------------+
| character_set_connection                      | utf8            |
| collation_connection                          | utf8_general_ci |
| connect_timeout                               | 10              |
| disconnect_on_expired_password                | ON              |
| extra_max_connections                         | 1               |
| init_connect                                  |                 |
| max_connect_errors                            | 100             |
| max_connections                               | 300             |
| max_user_connections                          | 0               |
| performance_schema_session_connect_attrs_size | 512             |
+-----------------------------------------------+-----------------+
10 rows in set (0.91 sec)

show variables like "key_buffer_size";
show variables like "query_cache_size";
show variables like "tmp_table_size";
show variables like "innodb_buffer_pool_size";
show variables like "innodb_additional_mem_pool_size";
show variables like "innodb_log_buffer_size";
show variables like "max_connections";
show variables like "sort_buffer_size";
show variables like "read_buffer_size";
show variables like "read_rnd_buffer_size";
show variables like "join_buffer_size";
show variables like "thread_stack";
show variables like "binlog_cache_size";

innodb_read_io_threads : set to 8 (or 16 if you need faster reads)
innodb_writes_io_threads : set to 8 (or 16 if you need faster writes)
innodb_buffer_pool_instances : set it to 32 (number of cores)

show variables like "innodb_read_io_threads";
show variables like "innodb_buffer_pool_instances";
show variables like "slave_parallel_workers";
show variables like "slave_parallel_type";
show variables like "sync_binlog";
show variables like "innodb_flush_log_at_trx_commit";

SELECT @@GLOBAL.innodb_read_io_threads,
@@GLOBAL.innodb_buffer_pool_instances,
@@GLOBAL.slave_parallel_workers,
@@GLOBAL.slave_parallel_type,
@@GLOBAL.sync_binlog,
@@GLOBAL.innodb_flush_log_at_trx_commit,
@@GLOBAL.innodb_flush_log_at_timeout\G

STOP SLAVE;
SET GLOBAL slave_parallel_workers=16;
SET GLOBAL slave_parallel_type=LOGICAL_CLOCK;
SET GLOBAL sync_binlog=0;
SET GLOBAL innodb_flush_log_at_trx_commit=2;
SET GLOBAL innodb_flush_log_at_timeout=1800;
START SLAVE;

backup

*************************** 1. row ***************************
        @@GLOBAL.innodb_read_io_threads: 4
  @@GLOBAL.innodb_buffer_pool_instances: 1
        @@GLOBAL.slave_parallel_workers: 8
           @@GLOBAL.slave_parallel_type: DATABASE
                   @@GLOBAL.sync_binlog: 1
@@GLOBAL.innodb_flush_log_at_trx_commit: 1
   @@GLOBAL.innodb_flush_log_at_timeout: 1
1 row in set (0.04 sec)

*************************** 1. row ***************************
        @@GLOBAL.innodb_read_io_threads: 4
  @@GLOBAL.innodb_buffer_pool_instances: 1
        @@GLOBAL.slave_parallel_workers: 8
           @@GLOBAL.slave_parallel_type: LOGICAL_CLOCK
                   @@GLOBAL.sync_binlog: 0
@@GLOBAL.innodb_flush_log_at_trx_commit: 2
   @@GLOBAL.innodb_flush_log_at_timeout: 1800
1 row in set (0.00 sec)

Master: binlog_group_commit_sync_delay
SELECT @@GLOBAL.binlog_group_commit_sync_delay\G

mysql> SELECT @@GLOBAL.binlog_group_commit_sync_delay\G
*************************** 1. row ***************************
@@GLOBAL.binlog_group_commit_sync_delay: 0
1 row in set (0.00 sec)

SET GLOBAL binlog_group_commit_sync_delay=10000;


SLAVE:
SELECT @@GLOBAL.innodb_lru_scan_depth;
SET GLOBAL innodb_lru_scan_depth=256;

```

```md
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.0.0.21
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000022
          Read_Master_Log_Pos: 115676601
               Relay_Log_File: replica-relay-bin.000054
                Relay_Log_Pos: 63360694
        Relay_Master_Log_File: master-bin.000022
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 63360479
              Relay_Log_Space: 115677497
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 25337
                               26029
                               27108
                               27133
                               27455
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 373cd9b1-58c4-11ea-9320-5254008afee6
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Reading event from the relay log
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 373cd9b1-58c4-11ea-9320-5254008afee6:1-1303219
            Executed_Gtid_Set: 373cd9b1-58c4-11ea-9320-5254008afee6:1-1250321
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)

mysql> SHOW MASTER STATUS\G
*************************** 1. row ***************************
             File: master-bin.000022
         Position: 115718139
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 373cd9b1-58c4-11ea-9320-5254008afee6:1-1303261
1 row in set (0.00 sec)

mysql> 

```

### Error log

2020-03-28T13:34:56.336666Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 8822ms. The settings might not be optimal. (flushed=2000, during the time.)

Multi-threaded slave statistics for channel '': seconds elapsed = 546; events assigned = 5121; worker queues filled over overrun level = 0; waited due a Worker queue full = 0; waited due the total size = 0; waited at clock conflicts = 800374417000 waited (count) when Workers occupied = 0 waited when Workers occupied = 0
