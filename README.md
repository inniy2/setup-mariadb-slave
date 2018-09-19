## Setup mariadb slave
- - - -  
0. Test env setup
```bash
cd $git_home_vagrant/master && vagrant up && cd ../slave1 && vagrant up && cd ../slave2 && vagrant up

sed -i '' '/192.168.33/d' ~/.ssh/known_hosts

cd ~/conn

source test_host1.sh
source test_host2.sh
source test_host3.sh

ssh vagrant@$test_host

cd $git_home_vagrant/master && vagrant destroy -f && cd ../slave1 && vagrant destroy -f && cd ../slave2 && vagrant destroy -f
```

1. Percona toolkit installation
```bash
cd /tmp

wget https://repo.percona.com/apt/percona-release_0.1-6.$(lsb_release -sc)_all.deb

sudo dpkg -i percona-release_0.1-6.$(lsb_release -sc)_all.deb

sudo apt-get update

sudo apt-cache search percona
sudo apt-cache search qpress

sudo apt-get install percona-toolkit percona-xtrabackup-24 qpress
```


5. ssh-keygen  (On master)
```bash
testuser> ssh-keygen -f /home/testuser/.ssh/id_rsa -C testuser

testuser> touch ~/.ssh/authorized_keys

testuser> cat ~/.ssh/id_rsa.pub > ~/.ssh/authorized_keys

testuser> sudo chmod 755 ~/.ssh
testuser> sudo chmod 600 ~/.ssh/*

tar cvfPz /tmp/id_rsa_pair.tar.gz  -C ~/.ssh id_rsa  id_rsa.pub authorized_keys

testuser> vim ~/.ssh/config 
testuser> chmod 600 ~/.ssh/config
```
> ADD  
[ssh/config](ssh-config)


6. Scp and extract public key
```bash
export test_host=

scp /tmp/id_rsa_pair.tar.gz  $test_host:/tmp

testuser> tar xvfPz /tmp/id_rsa_pair.tar.gz -C ~/.ssh/
```

12. Add replication user on master  
```sql
mysql> use mysql;
mysql> CREATE USER 'replication_user'@'%' IDENTIFIED BY 'bigs3cret';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';

mysql> CREATE USER 'pt_user'@'%' IDENTIFIED BY 'bigs3cret';
mysql> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, PROCESS, REPLICATION SLAVE, SUPER ON *.* TO 'pt_user'@'%';

mysql> CREATE USER 'pt_user'@'localhost' IDENTIFIED BY 'bigs3cret';
mysql> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, PROCESS, REPLICATION SLAVE, SUPER ON *.* TO 'pt_user'@'localhost';

mysql> CREATE USER 'pt_user'@'127.0.0.1' IDENTIFIED BY 'bigs3cret';
mysql> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, PROCESS, REPLICATION SLAVE, SUPER ON *.* TO 'pt_user'@'127.0.0.1';
```


12. Configuration change  
```bash
mysql> use mysql;

mysql> set session sql_log_bin=0;

mysql> set global sync_binlog = 1;

mysql> set global innodb_flush_log_at_trx_commit =1 ;
```


12. Stop target mariadb && delete files (on slave)  
```bash
sudo systemctl stop mariadb

sudo tar cvfPz backup.tar.gz -C /var/lib mysql

sudo rm -rf /var/lib/mysql/*

sudo chmod 777 /var/lib/mysql
```

13. Take backup (On master)  
```bash
export test_host=172.31.32.196
sudo innobackupex \
--compress \
--compress-threads=4 \
--slave-info \
--user=root \
--stream=xbstream /var/lib/mysql | ssh testuser@$test_host "xbstream -x -C /var/lib/mysql"
```

14. Restore mariadb (on slave)
```bash
sudo innobackupex \
    --decompress \
    --parallel=4 \
    --tmpdir=/tmp /var/lib/mysql

sudo find /var/lib/mysql -name "*.qp" -exec rm {} \;

sudo innobackupex \
    --defaults-file=/etc/mysql/my.cnf \
    --use-memory=1G \
    --apply-log  /var/lib/mysql

sudo chmod 755 /var/lib/mysql && sudo chown -R mysql:mysql /var/lib/mysql
```

15. Setup replication  
```bash
sudo systemctl start mariadb

sudo cat /var/lib/mysql/xtrabackup_slave_info
sudo cat /var/lib/mysql/xtrabackup_binlog_info

mysql> CHANGE MASTER TO
  MASTER_HOST='172.31.36.135',
  MASTER_USER='replication_user',
  MASTER_PASSWORD='bigs3cret',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='mariadb-bin.000009',
  MASTER_LOG_POS=73338193,
  MASTER_CONNECT_RETRY=10;

  STOP SLAVE;
  SET GLOBAL gtid_slave_pos = '0-1-1830827';
  CHANGE MASTER TO master_use_gtid=slave_pos;
  START SLAVE;

```

16. pt-table-checksum  
```bash
sudo vim /etc/percona-toolkit/percona-toolkit.conf
```
> ADD  
[percona-toolkit.conf](percona-toolkit.conf)

```sql
mysql> create database percona;
mysql> use percona;
mysql> CREATE TABLE `dsns` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `parent_id` int(11) DEFAULT NULL,
  `dsn` varchar(255) NOT NULL,
  PRIMARY KEY (`id`)
);

mysql> INSERT INTO `dsns` (`id`,`parent_id`,`dsn`) values (1,1,"h=172.31.36.135,u=pt_user,p=bigs3cret,P=3306");
mysql> INSERT INTO `dsns` (`id`,`parent_id`,`dsn`) values (2,2,"h=172.31.32.196,u=pt_user,p=bigs3cret,P=3307");
```

```bash
export pt_master_host="127.0.0.1"

pt-table-checksum \
--config=/etc/percona-toolkit/percona-toolkit.conf \
--replicate percona.checksums \
--ignore-databases=mysql \
--host=$pt_master_host \
--recursion-method=dsn=D=percona,t=dsns

pt-table-checksum \
--config=/etc/percona-toolkit/percona-toolkit.conf \
--replicate=percona.checksums \
--replicate-check-only \
--ignore-databases=mysql  \
--host=$pt_master_host \
--recursion-method=dsn=D=percona,t=dsns
```

90. Result
```bash
Starting checksum ...

            TS ERRORS  DIFFS     ROWS  DIFF_ROWS  CHUNKS SKIPPED    TIME TABLE
09-19T11:25:42      0      3 11056045          0      37       0  87.906 alpha.beta
09-19T11:26:42      0      0        2          0       1       0  60.017 percona.dsns
```

```bash
Checking if all tables can be checksummed ...
Starting checksum ...

            TS ERRORS  DIFFS     ROWS  DIFF_ROWS  CHUNKS SKIPPED    TIME TABLE
09-19T11:30:33      0      3 11057786          0      38       0  85.151 alpha.beta
09-19T11:31:33      0      0        2          0       1       0  60.021 percona.dsns
```

```bash
Checking if all tables can be checksummed ...
Starting checksum ...
Differences on srv2
TABLE CHUNK CNT_DIFF CRC_DIFF CHUNK_INDEX LOWER_BOUNDARY UPPER_BOUNDARY
alpha.beta 2 -1 1 PRIMARY 1001 108529
alpha.beta 11 -1 1 PRIMARY 2819155 3148900
alpha.beta 20 0 1 PRIMARY 5774881 6102615

Differences on ip-172-31-32-196
TABLE CHUNK CNT_DIFF CRC_DIFF CHUNK_INDEX LOWER_BOUNDARY UPPER_BOUNDARY
alpha.beta 2 -1 1 PRIMARY 1001 108529
alpha.beta 11 -1 1 PRIMARY 2819155 3148900
alpha.beta 20 0 1 PRIMARY 5774881 6102615
```



91. pt-table-sync
```bash
#!
pt-table-sync \
--config=/etc/percona-toolkit/percona-toolkit.conf \
--print \
--replicate percona.checksums h=127.0.0.1  \
--dry-run

# !
pt-table-sync \
--config=/etc/percona-toolkit/percona-toolkit.conf \
--print \
--replicate percona.checksums h=127.0.0.1  \
--execute
```

```bash
Checking if all tables can be checksummed ...
Starting checksum ...
            TS ERRORS  DIFFS     ROWS  DIFF_ROWS  CHUNKS SKIPPED    TIME TABLE
09-19T12:37:59      0      0 11081917          0      36       0  87.178 alpha.beta
09-19T12:38:59      0      0        2          0       1       0  60.020 percona.dsns
```

```bash
testuser pts/1        172.31.36.135    Wed Sep 19 10:19 - 10:19  (00:00)
testuser pts/1        172.31.36.135    Wed Sep 19 10:19 - 10:19  (00:00)
testuser pts/1        172.31.40.219    Wed Sep 19 09:54 - 09:54  (00:00)
testuser pts/0        133.237.7.66     Wed Sep 19 09:21   still logged in

Wed Sep 19 12:40:13 UTC 2018
```

15:00
21:40