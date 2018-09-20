## Task1 ojbect: Setup Mariadb replication for SRV3.
- - - -  
### Task condition  
I treat SRV1 and SRV2 as production hosts.   
As for SRV3, I presume that it is one of the production hosts but It is not IN-SERVICE status.  
Since I do not have any test environment, I created my own with vagrant.  
Following are the versions of tools that I use to create a test environment.  
```
Vagrant 2.1.1
Virtual Box 5.2.12
CentOS 7.3 3.10.0-693.21.1.el7.x86_64
MariaDB 10.1.36
```


Percona toolkit  
The reasons for my choice to use Percona toolkit are Xtrabackup, pt-table-checksum, and pt-table-sync.  
Xtrabackup will allow us to take backup from MySQL and MariaDB while the instances are online status.  
pt-table-checksum and pt-table-sync will allow us to check data consistency and sync data between master and replicas while they are processing DMS queries.  


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


This task requires transport data from one host to another.  
I create a public key and deploy to the target hosts in order to ease the work.  
In reality, using ssh public authentication can be restricted by each companies security policy.  


2. ssh-keygen  (On master)
```bash
testuser> ssh-keygen -f /home/testuser/.ssh/id_rsa -C testuser

testuser> touch ~/.ssh/authorized_keys

testuser> cat ~/.ssh/id_rsa.pub > ~/.ssh/authorized_keys

testuser> chmod 755 ~/.ssh
testuser> chmod 600 ~/.ssh/*

testuser> tar cvfPz /tmp/id_rsa_pair.tar.gz  -C ~/.ssh id_rsa  id_rsa.pub authorized_keys

# Send tar to target hosts
testuser> export test_host=
testuser> scp /tmp/id_rsa_pair.tar.gz  $test_host:/tmp
```


3. Extract key (On slave)  
```bash
testuser> tar xvfPz /tmp/id_rsa_pair.tar.gz -C ~/.ssh/
```


4. ssh configuration  
```bash
testuser> vim ~/.ssh/config 
```
> vim ~/.ssh/config  
```
Host 172.31.x.x
    StrictHostKeyChecking no
```
```bash
testuser> chmod 600 ~/.ssh/config
```


In a real situation adding additional users should be consulted with the service owner.  
I try to use the percona toolkit with an empty password of 'root'@'localhost', however, it cause many problems with connection host with DNS config and it took me so much to time.  
I eventually decide to add additional users with a password to work it correctly.  
As for the existing replication user, I was not provided existing password nor could not find master.info file which should contain plain text password from replication user password, I created additional replication user to connect from SVR3 to SVR2.  


5. Add additional users to instances  
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

It is a strong recommandation from MySQL to set sync_binlog = 1 and innodb_flush_log_at_trx_commit = 1 if any service running with replicas.  


6. Configuration change  
```bash
mysql> use mysql;

mysql> set session sql_log_bin=0;

mysql> set global sync_binlog = 1;

mysql> set global innodb_flush_log_at_trx_commit =1 ;
```


7. Backup data && delete data (On the target slave)  
```bash
# backup existing data just in case
sudo tar cvfPz backup.tar.gz -C /var/lib mysql

sudo rm -rf /var/lib/mysql/*

sudo chmod 777 /var/lib/mysql
```

There are several ways to take backup with Xtrabackup.  
I choose stream backup because it has fewer steps to restore part and it does not require disk space on the master and it is the fastest option overall if you have enough bandwidth in network resource.  
If you have limited bandwidth in your network, you can use PV to control the network traffic.  


8. Take backup (On master)  
```bash
export test_host=172.31.32.196

sudo innobackupex \
--compress \
--compress-threads=4 \
--slave-info \
--user=root \
--stream=xbstream /var/lib/mysql | ssh testuser@$test_host "xbstream -x -C /var/lib/mysql"
```


9. Restore MariaDB (On the target slave)
```bash
# Uncompress
sudo innobackupex \
    --decompress \
    --parallel=4 \
    --tmpdir=/tmp /var/lib/mysql

# Delete compressed files
sudo find /var/lib/mysql -name "*.qp" -exec rm {} \;

# Apply log
sudo innobackupex \
    --defaults-file=/etc/mysql/my.cnf \
    --use-memory=1G \
    --apply-log  /var/lib/mysql

# Change owner back to mysql
sudo chmod 755 /var/lib/mysql && sudo chown -R mysql:mysql /var/lib/mysql
```


10. Setup replication  
```bash
sudo systemctl start mariadb

# Read replication information from following files
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

  SET GLOBAL gtid_slave_pos = '0-1-1830827';
  CHANGE MASTER TO master_use_gtid=slave_pos;
  START SLAVE;
```


pt-table-checksum and pt-table-sync will allow us to check data consistency and sync data between master and replicas while they are processing DMS queries.  


11. Config percona-toolkit.conf
```bash
sudo vim /etc/percona-toolkit/percona-toolkit.conf
```
> sudo vim /etc/percona-toolkit/percona-toolkit.conf  
```
user=pt_user
password=bigs3cret
```


12. Create dsn information table
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


13. Check data consistency with pt-table-checksum
```bash
export pt_master_host="127.0.0.1"

pt-table-checksum \
--config=/etc/percona-toolkit/percona-toolkit.conf \
--replicate percona.checksums \
--ignore-databases=mysql \
--host=$pt_master_host \
--recursion-method=dsn=D=percona,t=dsns
```
> Result:  3 DIFFS in beta table in aplha database  
```
Checking if all tables can be checksummed ...
Starting checksum ...

            TS ERRORS  DIFFS     ROWS  DIFF_ROWS  CHUNKS SKIPPED    TIME TABLE
09-19T11:30:33      0      3 11057786          0      38       0  85.151 alpha.beta
09-19T11:31:33      0      0        2          0       1       0  60.021 percona.dsns
```


With --replicate-check-only option, you can find the difference in host level.  

14. Check status of difference
```bash
pt-table-checksum \
--config=/etc/percona-toolkit/percona-toolkit.conf \
--replicate=percona.checksums \
--replicate-check-only \
--ignore-databases=mysql  \
--host=$pt_master_host \
--recursion-method=dsn=D=percona,t=dsns
```
> Result:  SRV2 and SRV3 are all different from SRV1  
```
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

To correct the data, you can use pt-table-sync.  
pt-table-sync offer many different options to make sync data again, however, to find the correct option you will need test in each case carefully because same options can break your replication in some cases.  
It is a strong recommendation to use --dry-run before --execute.


15. pt-table-sync
```bash
# --dry-run
pt-table-sync \
--config=/etc/percona-toolkit/percona-toolkit.conf \
--print \
--replicate percona.checksums h=127.0.0.1  \
--dry-run

# --execute
pt-table-sync \
--config=/etc/percona-toolkit/percona-toolkit.conf \
--print \
--replicate percona.checksums h=127.0.0.1  \
--execute
```
> Result: Tables are now sync.
```bash
Checking if all tables can be checksummed ...
Starting checksum ...
            TS ERRORS  DIFFS     ROWS  DIFF_ROWS  CHUNKS SKIPPED    TIME TABLE
09-19T12:37:59      0      0 11081917          0      36       0  87.178 alpha.beta
09-19T12:38:59      0      0        2          0       1       0  60.020 percona.dsns
```


It roughly took me 4H 30M to create the test environment and up and running the SVR3.  
The most time-consuming work ( more then 2 hours ) was because of the empty password of root users, I found difficult to connect all the replicas with the percona toolkit.  
Any other steps only took me around 10 to 30 minutes each.  
I considered the hosts as real production hosts so I tried to minimize modification of the current setup.  


16. Working hours  
```bash
testuser pts/1        172.31.36.135    Wed Sep 19 10:19 - 10:19  (00:00)
testuser pts/1        172.31.36.135    Wed Sep 19 10:19 - 10:19  (00:00)
testuser pts/1        172.31.40.219    Wed Sep 19 09:54 - 09:54  (00:00)
testuser pts/0        133.237.7.66     Wed Sep 19 09:21   still logged in

exit time: Wed Sep 19 12:40:13 UTC 2018
```
