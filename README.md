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

1. OS setup  
```bash
sudo yum install -y wget vim screen

cd /tmp

sudo wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

sudo rpm -ivh /tmp/epel-release-latest-7.noarch.rpm 

sudo wget http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm

sudo yum -y install http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm 
```

2. Copy percona-release.repo  
```bash
sudo vi /etc/yum.repos.d/percona-release.repo
```
> ADD  
[percon-release.repo](percona-release-enable.repo)  

3. Install additional pakages
```bash
sudo yum -y install percona-toolkit.x86_64 percona-xtrabackup-24.x86_64 qpress.x86_64
```

4. If Additional OS user is required  
```bash
sudo groupadd sangsunbae -g 57000
sudo useradd sangsunbae  -u 57000 -g 57000 -G sangsunbae -d /home/sangsunbae -m -s /bin/bash -c 'admin user'

sudo touch  /etc/sudoers.d/sangsunbae
sudo visudo -f /etc/sudoers.d/sangsunbae
```
> ADD
[sangsunbae](sangsunbae)


5. ssh-keygen  (On master)
```bash
sangsunbae> ssh-keygen -f /home/sangsunbae/.ssh/id_rsa -C sangsunbae

sangsunbae> touch ~/.ssh/authorized_keys

sangsunbae> cat ~/.ssh/id_rsa.pub > ~/.ssh/authorized_keys

sangsunbae> sudo chmod 755 ~/.ssh
sangsunbae> sudo chmod 600 ~/.ssh/*

tar cvfPz /tmp/id_rsa_pair.tar.gz  -C ~/.ssh id_rsa  id_rsa.pub authorized_keys

sangsunbae> vim ~/.ssh/config && chmod 600 ~/.ssh/config
```
> ADD  
[ssh/config](ssh-config)


6. Scp and extract public key
```bash
export test_host=

scp /tmp/id_rsa_pair.tar.gz  vagrant@$test_host:/tmp

sangsunbae> tar xvfPz /tmp/id_rsa_pair.tar.gz -C ~/.ssh/
```

7. Disable percona-release.repo
```bash
sudo vim /etc/yum.repos.d/percona-release.repo
```
> ADD  
[percon-release.repo](percona-release-disable.repo)  

8. MariaDB repo
```bash
sudo vim /etc/yum.repos.d/mariadb.repo 
```
> ADD  
[mariadb.repo](mariadb.repo)

9. Install mariadb  
```bash
sudo yum -y install MariaDB-server MariaDB-client
```

10. Set my.cnf  
```bash
sudo vim /etc/my.cnf
```
> ADD  
[master](master.cnf)
[slave1](slave1.cnf)
[slave2](slave2.cnf)

11. Start mariadb
```bash
sudo mkdir /var/log/mariadb && sudo touch /var/log/mariadb/mariadb.log
sudo chown -R mysql:mysql /var/log/mariadb

sudo systemctl start mariadb
sudo mysql_secure_installation 
```

12. Add replication user on master  
```sql
mysql> use mysql;
mysql> CREATE USER 'replication_user'@'%' IDENTIFIED BY 'bigs3cret';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';

mysql> CREATE USER 'pt_user'@'%' IDENTIFIED BY 'bigs3cret';
mysql> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, PROCESS, REPLICATION SLAVE, SUPER ON *.* TO 'pt_user'@'%';
```

12. Stop target mariadb && delete files (on slave)  
```bash
sudo systemctl stop mariadb

sudo rm -rf /var/lib/mysql/*

sudo chmod 777 /var/lib/mysql
```

13. Take backup (On master)  
```bash
export test_host=
sudo innobackupex \
--compress \
--compress-threads=4 \
--slave-info \
--user=root \
--password=test\!\! \
--stream=xbstream /var/lib/mysql | ssh sangsunbae@$test_host "xbstream -x -C /var/lib/mysql"
```

14. Restore mariadb (on slave)
```bash
sudo innobackupex \
    --decompress \
    --parallel=4 \
    --tmpdir=/tmp /var/lib/mysql

sudo find /var/lib/mysql -name "*.qp" -exec rm {} \;

sudo innobackupex \
    --defaults-file=my.cnf \
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
  MASTER_HOST='192.168.33.11',
  MASTER_USER='replication_user',
  MASTER_PASSWORD='bigs3cret',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='mariadb-bin.000002',
  MASTER_LOG_POS=315,
  MASTER_CONNECT_RETRY=10;
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

mysql> INSERT INTO `dsns` (`id`,`parent_id`,`dsn`) values (1,1,"h=192.168.33.11,u=pt_user,p=bigs3cret,P=3306");
mysql> INSERT INTO `dsns` (`id`,`parent_id`,`dsn`) values (2,2,"h=192.168.33.12,u=pt_user,p=bigs3cret,P=3306");
```

```bash
export pt_master_host="192.168.33.10"

pt-table-checksum \
--config=/etc/percona-toolkit/percona-toolkit.conf \
--replicate percona.checksums \
--ignore-databases=mysql \
--host=$pt_master_host \
--recursion-method=dsn=D=percona,t=dsns

```