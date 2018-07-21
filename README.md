
# Configuração do servidor Master:

```
Partimos do principio que o PostgreSQL está instalado na versão desejada.
```

## Criar Pasta do archive

```
cd /var/lib/pgsql/data

[root@centos data]# mkdir -p archive

[root@centos data]# chmod 700 /var/lib/pgsql/data/archive/

[root@centos data]# chown -R postgres:postgres /var/lib/pgsql/data/archive/
```

## Acessar o diretorio /var/lib/pgsql/data e mudar parametros no arquivo de configuração postgresql.conf:

```
[root@centos data]# cd /var/lib/pgsql/data

[root@centos data]#  nano postgresql.conf
```

## Mudar os campos abaixo:

```
listen_addresses = '*'

wal_level = hot_standby

synchronous_commit = local

archive_mode = on

archive_command = 'cp %p /var/lib/postgresql/9.6/main/archive/%f'

max_wal_senders = 2

wal_keep_segments = 10

synchronous_standby_names = 'pgslave001'
```

## Acessar o diretorio /var/lib/pgsql/data e adiconar as linhas no final do arquivo de configuração pg_hba.conf:

```
#Localhost
host    replication     replica          127.0.0.1/32            md5   #IP Servidor Local

#PostgreSQL Master IP address
host    replication     replica          0.0.0.0/32              md5   #Substituir o campo "0.0.0.0/32" pelo IP do Servidor Primário.

#PostgreSQL Slave IP address
host    replication     replica          0.0.0.0/32              md5   #Substituir o campo "0.0.0.0/32" pelo IP do Servidor Secundário.
```

## Efetuar restart no servidor Primário:

```
[root@centos ~]# sudo systemctl restart postgresql
```

## Acessar o banco de dados master:

```
[root@db-atendimento ~]# su - postgres
Last login: Wed Jul 11 14:17:32 -03 2018 on pts/0
-bash-4.2$ psql
psql (9.2.23)
Type "help" for help.

postgres=# 
```

## Criar um usuário e senha para replica:

```
CREATE USER replica REPLICATION LOGIN ENCRYPTED PASSWORD 'aqwe123@';
```

#### Configuração do servidor Primário completa.   

```
```

# Configuração do servidor Secundário:

```
Partimos do principio que o PostgreSQL está instalado na versão desejada igual ao Primário.
```


## Parar o serviço de banco de dados postgresql:

```
sudo systemctl stop postgresql
```

## Copiar dados do servidor Primário para o servidor Secundário.

```
pg_basebackup -h "IP-To-MASTER" -U replica -D /var/lib/pgsql/data/ -P --xlog
```

## Acessar o diretorio /var/lib/pgsql/data e mudar parametros no arquivo de configuração postgresql.conf:

```
listen_addresses = '*'

wal_level = hot_standby

synchronous_commit = local

max_wal_senders = 2

wal_keep_segments = 10

synchronous_standby_names = 'pgslave001'

hot_standby = on

max_standby_streaming_delay = -1

hot_standby_feedback = on 
```

#### Salve o arquivo.


## Acessar o diretorio /var/lib/pgsql/data e criar o arquivo recovery.conf:

```
----------------------------------------
standby_mode = 'on'

primary_conninfo = 'host=ip-to-master port=5432 user=replica password=aqwe123@ application_name=pgslave001'

restore_command = 'cp /var/lib/pgsql/data/archive/%f %p'

trigger_file = '/tmp/postgresql.trigger.5432'
----------------------------------------
```
## Definir as devidas permissões no arquivo.

```
chmod 600 recovery.conf

chown -R  postgres:postgres /var/lib/pgsql

sudo systemctl restart postgresql
```

#### Trasferencia de dados e configuração do PostgreSQL secundário completa.

# Agora vamos testar

#### Para testar, verificaremos o status de replicação do PostgreSQL 9.6 e tentaremos criar uma nova tabela no servidor Primário, depois validaremos a replicação verificando todos os dados do servidor Secundário.

#### Faça o login no servidor Primário como usuário postgres.

```
su - postgres
```

#### Faça os testes abaixo para validar a replica do PostgreSQL.


```
-bash-4.2$  psql -c "select application_name, state, sync_priority, sync_state from pg_stat_replication;"

[root@centos ~]# su - postgres

Last login: Wed Jul 11 14:03:07 -03 2018 on pts/0

-bash-4.2$ psql -c "select application_name, state, sync_priority, sync_state from pg_stat_replication;"
 application_name |   state   | sync_priority | sync_state 
------------------+-----------+---------------+------------
 pgslave001       | streaming |             1 | sync
(1 row)
```
```
-bash-4.2$  psql -x -c "select * from pg_stat_replication;"

-bash-4.2$ psql -x -c "select * from pg_stat_replication;"
-[ RECORD 1 ]----+------------------------------
pid              | 14575
usesysid         | 541135
usename          | replica
application_name | pgslave001
client_addr      | 138.121.71.143
client_hostname  | 
client_port      | 54386
backend_start    | 2018-07-11 10:39:24.885267-04
state            | streaming
sent_location    | 16/F70010C0
write_location   | 16/F70010C0
flush_location   | 16/F70010C0
replay_location  | 16/F70010C0
sync_priority    | 1
sync_state       | sync

```


#### Em seguida, crie uma nova tabela no servidor Primário, faça login com usuário postgres no servidor Primário.

```
[root@centos tmp]# su - postgres

Last login: Wed Jul 11 14:18:13 -03 2018 on pts/0

-bash-4.2$ psql

psql (9.2.23)
Type "help" for help.

postgres=# 

```

```
postgres=#  CREATE TABLE replica_test (hakase varchar(100));
CREATE TABLE

postgres=#  INSERT INTO replica_test VALUES ('howtoforge.com');
INSERT 0 1

postgres=#  INSERT INTO replica_test VALUES ('This is from Master');
INSERT 0 1

postgres=#  INSERT INTO replica_test VALUES ('pg replication by hakase-labs');
INSERT 0 1

postgres=# select * from replica_test;
            hakase             
-------------------------------
 howtoforge.com
 This is from Master
 pg replication by hakase-labs
(3 rows)

```

#### Em seguida, faça login no usuário postgres no servidor Secundário e acesse o terminal psql.


```
[root@centos tmp]# su - postgres

Last login: Wed Jul 11 14:18:13 -03 2018 on pts/0

-bash-4.2$ psql

psql (9.2.23)
Type "help" for help.

postgres=# 
```

#### Verifique os dados na tabela 'replica_test' com a consulta postgres abaixo.

```
postgres=# select * from replica_test;
            hakase             
-------------------------------
 howtoforge.com
 This is from Master
 pg replication by hakase-labs
(3 rows)

```

### Teste adicional:

#### Teste para escrever no servidor Secundário com a consulta abaixo.

```
postgres=# INSERT INTO replica_test VALUES ('this is SLAVE');

ERROR:  cannot execute INSERT in a read-only transaction
```

