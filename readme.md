
# Replicação MySQL ~~master-slave~~ source-replica (Ubuntu Desktop 22.04 LTS)

Esse tutorial tem como referência o tutorial da [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-set-up-replication-in-mysql).

Utilizaremos prefixos de usuários para identificar em qual máquina estamos trabalhando. O prefixo **source** será utilizado para a máquina que será o servidor principal, e o prefixo **replica** será utilizado para a máquina que será o servidor secundário. Lembre de removê-los quando for utilizar os comandos.

## Caso precise recomeçar
Caso você tenha se perdido no processo, ou queira recomeçar, você pode deletar tudo e começar do zero. Para isso, basta desistalar o mysql e deletar os arquivos de configuração.
```bash
replica:~$ sudo apt remove --purge mysql-server -y
```
```bash
source:~$ sudo apt remove --purge mysql-server -y
```
Quando eu reinstalei o mysql, ele ainda pareceu utilizar as configurações antigas, então eu tive que fazer uma limpeza completa deletar alguns arquivos que restaram utilizando:
```bash
replica:~$ sudo find / -type d -name "*mysql*" -exec rm -r {} +
```
```bash
source:~$ sudo find / -type d -name "*mysql*" -exec rm -r {} +
```
---
Esse tutorial foi feito utilizando o Ubuntu 22.04.3 LTS versão **Desktop** em [VirtualBox](https://www.virtualbox.org/).
## Preparativos nas duas máquinas
Como utilizaremos duas máquinas virtuais, para facilitar a transição entre uma máquina e outra, vamos utilizar o ssh para trabalhar apenas em uma máquina por vez. Para isso, vamos instalar o ssh nas duas máquinas.
```bash
 sudo apt install openssh-server -y
```

## Instalação
### Instalando o MySQL (replica)
```bash
replica:~$ sudo apt update
```
```bash
replica:~$ sudo apt install mysql-server -y
```
### Instalando o MySQL (source)
```bash
source:~$ sudo apt update
```
```bash
source:~$ sudo apt install mysql-server -y
```

## Configurando o banco de dados (source)
### Configurando o MySQL
Primeiro precisamos obter o endereço IP da máquina replica (certifique-se que ambas as máquinas estão no modo bridge, caso precise mudar veja [como colocar máquina no Virtual Box no modo Bridge](https://pplware.sapo.pt/tutoriais/virtualbox-ligar-diretamente-maquina-rede/#:~:text=No%20VirtualBox%2C%20basta%20que%20carreguem,a%20placa%20para%20modo%20Bridge.)). Para isso, basta abrir um novo terminal (utilizando o atalho `ctrl`+ `alt` + `t`) e digitar:
```bash
replica:~$ ip a
```
Você verá algo como:
```txt
replica:~$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:0a:b8:82 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.56/24 brd 192.168.0.255 scope global dynamic noprefixroute enp0s3
       valid_lft 85667sec preferred_lft 85667sec
    inet6 2804:1e90:121b:3b00:5b48:f99:7d72:aafa/64 scope global temporary dynamic 
       valid_lft 86198sec preferred_lft 85345sec
    inet6 2804:1e90:121b:3b00:7c08:81db:2c3b:5076/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 86198sec preferred_lft 86198sec
    inet6 fe80::635c:665c:f81a:ac33/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```
Em seguida, procure pela linha que contém `enp0s3` (ou algo parecido) e copie o endereço IP que está na linha `inet`. No meu caso, o endereço IP é `192.168.0.56`. Faça esse processo na máquina replica também e guarde o endereço IP da máquina replica.

```bash
source:~$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
Nesse arquivo, primeiro iremos alterar o endereço IP que o MySQL irá escutar. Para isso, iremos procurar pela linha que contém `bind-address` e alterar o endereço IP para o endereço IP da máquina source.
```txt
...
bind-address            = 127.0.0.1
...
```
Para:
```txt
...
bind-address            = source_ip
...
```
> **Atenção:** Não esqueça de substituir `source_ip` pelo endereço IP da máquina source.
Descomente a linha que contém `server-id` e altere o valor para `1`.
```txt
...
# server-id               = 1
...
```
Para:
```txt
...
server-id               = 1
...
```
Descomente a linha que contém `log_bin`.
```txt
...
# log_bin                 = /var/log/mysql/mysql-bin.log
...
```
Para:
```txt
...
log_bin                 = /var/log/mysql/mysql-bin.log
...
```
Salve o arquivo e feche-o, pra isso, utilize o atalho `ctrl` + `x`, em seguida digite `y` e pressione `enter`.

Criando o usuário que será utilizado para a replicação.
```bash
source:~$ sudo mysql
```
```sql
msql> CREATE USER 'replica_user'@'replica_ip' IDENTIFIED WITH mysql_native_password BY 'password';
```
> **Atenção:** Não esqueça de remover o prrefixo `mysql>`.<br>
> **Atenção:** Não esqueça de substituir `replica_ip` pelo endereço IP da máquina replica.<br>

```sql
msql> GRANT REPLICATION replica ON *.* TO 'replica_user'@'replica_ip';
```
> **Atenção:** Não esqueça de substituir `replica_ip` pelo endereço IP da máquina replica.<br>
```sql	
msql> FLUSH PRIVILEGES;
```

### Recuperando o binlog do source
```sql
msql> SHOW source STATUS;
```
Output:
```txt
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
Guarde essas informações, pois iremos utilizá-las na configuração da réplica.

## Configurando o banco de dados (replica)
Como dito anteriormente, vamos modificar a máquina de réplica direto da máquina source utilizando o ssh.
```bash
source:~$ ssh replica_user@replica_ip
```
> **Atenção:** Não esqueça de substituir `replica_user` pelo nome do usuário que você criou na máquina réplica.<br>
> **Atenção:** Não esqueça de substituir `replica_ip` pelo endereço IP da máquina replica.<br>
> **Atenção:** Caso seja a primeira vez que você está se conectando na máquina replica, você receberá uma mensagem de erro, basta digitar `yes` e pressionar `enter`, em seguida digite a senha da máquina replica e pressione `enter`.

Para configurar a máquina replica, vamos fazer um processo parecido com o que fizemos na máquina source.
```bash
replica:~$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
Nesse arquivo, não precisamos alterar o endereço IP que o MySQL irá escutar. Então, descomente a linha que contém `server-id` e altere o valor para `2`.
```txt
...
# server-id               = 1
...
```
Para:
```txt
...
server-id               = 2
...
```
Descomente a linha que contém `log_bin`.
```txt
...
# log_bin                 = /var/log/mysql/mysql-bin.log
...
```
Para:
```txt
...
log_bin                 = /var/log/mysql/mysql-bin.log  
...
```
Adicione o `relay_log` no final do arquivo.
```txt
...
relay_log               = /var/log/mysql/replica-relay-bin.log
```

Salve o arquivo e feche-o, pra isso, utilize o atalho `ctrl` + `x`, em seguida digite `y` e pressione `enter`.
Restarte o MySQL.
```bash
replica:~$ sudo systemctl restart mysql
```

### Iniciando replicação
```bash
replica:~$ sudo mysql
```
```sql
msql> STOP replica;
```
```sql
mysql> CHANGE REPLICATION SOURCE TO
SOURCE_HOST='source_ip',
SOURCE_USER='replica_user',
SOURCE_PASSWORD='password',
SOURCE_LOG_FILE='mysql-bin.000001',
SOURCE_LOG_POS=899;
```
> **Atenção:** Não esqueça de substituir `source_ip` pelo endereço IP da máquina source.<br>
> **Atenção:** Não esqueça de substituir `mysql-bin.000001` pelo valor que você obteve no comando `SHOW source STATUS;`.<br>
> **Atenção:** Não esqueça de substituir `899` pelo valor que você obteve no comando `SHOW source STATUS;`.<br>

```sql
mysql> START REPLICA;
```
```sql
mysql> SHOW REPLICA STATUS \G
```
Output:
```txt
*************************** 1. row ***************************
               replica_IO_State: Waiting for source to send event
                  source_Host: source_ip
                  source_User: replica_user
                  source_Port: 3306
                Connect_Retry: 60
              source_Log_File: mysql-bin.000001
          Read_source_Log_Pos: 899
               Relay_Log_File: replica-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_source_Log_File: mysql-bin.000001
             replica_IO_Running: Yes
            replica_SQL_Running: Yes
            Replicate_Do_DB: 
        Replicate_Ignore_DB: 
         Replicate_Do_Table: 
     Replicate_Ignore_Table: 
    Replicate_Wild_Do_Table:
...
```

## Testando a replicação
```bash
source:~$ sudo mysql
```
```sql
mysql> CREATE DATABASE test;
```
```sql
mysql> SHOW DATABASES;
```
Output:
```txt
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.00 sec)
```
Agora vamos verificar se o banco de dados foi criado na máquina replica.
```bash
replica:~$ sudo mysql
```
```sql
mysql> SHOW DATABASES;
```
Output:
```txt
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.00 sec)
```
Pronto! Agora você tem um banco de dados replicado entre duas máquinas virtuais.

## Caso mude de rede
Caso você mude de rede, você precisará alterar o endereço IP da máquina source no arquivo de configuração da máquina source.
```bash
source:~$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
Nesse arquivo, procure pela linha que contém `bind-address` e altere o endereço IP para o endereço IP da máquina source.
```txt
...
bind-address            = antigo_source_ip
...
```
Para:
```txt
...
bind-address            = novo_source_ip
...
```
> **Atenção:** Não esqueça de substituir `novo_source_ip` pelo novo endereço IP da máquina source.

Em seguida, reinicie o MySQL.
```bash
source:~$ sudo systemctl restart mysql
```
O arquivo de log irá mudar, então precisamos capturar novamente essa informação.
```bash
source:~$ sudo mysql
```
```sql
msql> SHOW source STATUS;
```
Output:
```txt
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
Guarde essas informações, pois iremos utilizá-las na configuração da réplica.


Também precisamos criar um novo usuário para a máquina replica.
```sql
msql> CREATE USER 'replica_user'@'novo_replica_ip' IDENTIFIED WITH mysql_native_password BY 'password';
```
> **Atenção:** Não esqueça de substituir `novo_replica_ip` pelo novo endereço IP da máquina replica.

```sql
msql> GRANT REPLICATION replica ON *.* TO 'replica_user'@'novo_replica_ip';
```
```sql
msql> FLUSH PRIVILEGES;
```

Agora precisamos atualizar as informações do source na máquina replica.
```bash
replica:~$ sudo mysql
```
```sql
msql> STOP REPLICA;
```
```sql
mysql> CHANGE REPLICATION SOURCE TO
SOURCE_HOST='novo_source_ip',
SOURCE_USER='replica_user',
SOURCE_PASSWORD='password',
SOURCE_LOG_FILE='mysql-bin.000002',
SOURCE_LOG_POS=899;
```
> **Atenção:** Não esqueça de substituir `novo_source_ip` pelo novo endereço IP da máquina source.<br>
> **Atenção:** Não esqueça de substituir `mysql-bin.000002` pelo valor que você obteve no comando `SHOW source STATUS;`.<br>
> **Atenção:** Não esqueça de substituir `899` pelo valor que você obteve no comando `SHOW source STATUS;`.

```sql
mysql> START REPLICA;
```
```sql
mysql> SHOW REPLICA STATUS \G
```
Output:
```txt
*************************** 1. row ***************************
               replica_IO_State: Waiting for source to send event
                  source_Host: novo_source_ip
                  source_User: replica_user
                  source_Port: 3306
                Connect_Retry: 60
              source_Log_File: mysql-bin.000002
          Read_source_Log_Pos: 899
               Relay_Log_File: replica-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_source_Log_File: mysql-bin.000002
             replica_IO_Running: Yes
            replica_SQL_Running: Yes
            Replicate_Do_DB: 
        Replicate_Ignore_DB: 
         Replicate_Do_Table: 
     Replicate_Ignore_Table: 
    Replicate_Wild_Do_Table:
...
```