## Task2 object: Do you see any problems with the setup, security or otherwise?  
- - - -  
## Task condition  
I treat the SRV[1-3] host as production hosts.  
Some countermeasures for security vulnerability like changing port and setup password and deleting users should be approached carefully because it will cause some service impact.  
Usually, these changes will require discussion with the service owner and developers.  


1. Empty password and unnecessary users.  
> I found root user does not have a password.  
> It is a strong recommendation to setup up a password or even delete root user.  
> MySQL and MariaDB support similar approch running a `mysql_secure_installation`  
```
bash> mysql_secure_installation
```
```
Example:
localhost:# mysql_secure_installation 


NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!


In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.


Enter current password for root (enter for none): 
OK, successfully used password, moving on...


Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorization.


You already have a root password set, so you can safely answer 'n'.


Change the root password? [Y/n] n
 ... skipping.


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.


Remove anonymous users? [Y/n] y
 ... Success!


Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.


Disallow root login remotely? [Y/n] y
 ... Success!


By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.


Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!


Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.


Reload privilege tables now? [Y/n] y
 ... Success!


Cleaning up...


All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.


Thanks for using MariaDB!
```


2. Access host restriction.  
> I found 'xxx'@'%' type of users and it is not recommended.  
> I also created 'pt_user'@'%' to ease the work, however, we do not use this type of users in the production.  


3. Using known port  
> SVR1 and SVR2 are running on port 3306 which is known port for MySQL and MariaDB.  
> It is also a strong recommendation to use a different port for production usages.  


4. Enable log_error && set log_warnings = 1  
> It will show you critical information like access denied errors, slave reconnections, killed slaves, error reading relay logs. This information will be useful for security perspective and replication SLA.  
> It will generate lots of logs so it is better to use with the logrotate.


5. Using audit plugin  
> Unlike MySQL, MariaDB supports an audit plugin and you do not need to buy an enterprise edition.  
> I found it in /usr/lib/mysql/plugin/server_audit.so  


6. set local_infile = 0  
> It is wildly known that enabling local_infile may cause a security vulnerability.  
> It is recommended to disable this feature but some services heavily depend on the function and sometimes it is difficult to disable it.  


7. Replication data consistency 
> sync_binlog = 1 and innodb_flush_log_at_trx_commit = 1 is strongly recommended for services which uses replicas and if the service consider the data consistency importantly.  
> These options, I changed them while I was creating SVR3 because it was important to replicas.  
> Also for replicas, it is better to set `READ_ONLY` option for keeping the data consistency.  


8. Utilizing more IO capacity  
> Disk information shows that the MariaDB tablespaces are located in /dev/nvme0n1p1.  
> Judging by the mapping name it is an NVMe SSD and if this is a real NVMe attached to PCI-Express, then we can utilize more IO capacity.  
> Current setup is `innodb_io_capacity = 200` and `innodb_io_capacity_max = 2000`, however, I believe there is more room for the capacity.


9. Disk mount option  
> The current setup of /dev/vnme01np1 is ext4 with the barrier.  
> I found xfs is a little bit better than ext4 in MySQL case.  
> It may depends on the brand of NVMe.  
> With a certain product with a certain firmware, nobarrier option will increase write performance by more than 6 times.  

