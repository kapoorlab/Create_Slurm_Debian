# Create_Slurm_Debian
Create a Slurm machine on Debian

## Overview



Slurm overview: https://slurm.schedmd.com/overview.html

> Slurm is an open source, fault-tolerant, and highly scalable cluster management and job scheduling system for large and small Linux clusters. Slurm requires no kernel modifications for its operation and is relatively self-contained. As a cluster workload manager, Slurm has three key functions. First, it allocates exclusive and/or non-exclusive access to resources (compute nodes) to users for some duration of time so they can perform work. Second, it provides a framework for starting, executing, and monitoring work (normally a parallel job) on the set of allocated nodes. Finally, it arbitrates contention for resources by managing a queue of pending work. Optional plugins can be used for accounting, advanced reservation, gang scheduling (time sharing for parallel jobs), backfill scheduling, topology optimized resource selection, resource limits by user or bank account, and sophisticated multifactor job prioritization algorithms.


# Run the commands as non priviliged user
## Install slurm and associated components on slurm controller node.
Install prerequisites 

Debian GNU/Linux 11 (bullseye)
```console
$ apt-get update
$ apt-get install git gcc make ruby ruby-dev libpam0g-dev libmariadb-dev-compat libmariadb-dev
$ gem install fpm
```


### Install munge
MUNGE (MUNGE Uid 'N' Gid Emporium) is an authentication service for creating and validating credentials.
https://dun.github.io/munge/


```console
$ wget https://github.com/dun/munge/releases/download/munge-0.5.15/munge-0.5.15.tar.xz
$ tar xJf munge-0.5.15.tar.xz
$ cd munge-0.5.15
$ ./configure \
   --prefix=/usr \
   --sysconfdir=/etc \
   --localstatedir=/var \
   --runstatedir=/run
$ make
$ make check
$ sudo make install
$ sudo systemctl enable munge.service
$ sudo systemctl start munge.service
The munge.key file is located in /etc/munge/munge.key and has the UID and GID of the nologin user munge


```


### Test munge
```console
$ munge -n | unmunge
(base) debian@kapoorlabs:~$ munge -n | unmunge
STATUS:          Success (0)
ENCODE_HOST:     kapoorlabs.fr (51.210.158.171)
ENCODE_TIME:     2023-03-16 18:45:14 +0000 (1678992314)
DECODE_TIME:     2023-03-16 18:45:14 +0000 (1678992314)
TTL:             300
CIPHER:          aes128 (4)
MAC:             sha256 (5)
ZIP:             none (0)
UID:             debian (1000)
GID:             debian (1000)
LENGTH:          0
```

### Install MariaDB for Slurm accounting
MariaDB is an open source Mysql compatible database.
https://mariadb.org/

In the following steps change the DB password "slurmdbpass" to something secure.


```console
$ apt-get install mariadb-server
$ systemctl enable mysql
$ systemctl start mysql
$ mysql -u root
create database slurm_acct_db;
create user 'debian'@'kapoorlabs.fr';
set password for 'debian'@'kapoorlabs.fr' = password('slurmdbpass');
grant usage on *.* to 'debian'@'kapoorlabs.fr';
grant all privileges on slurm_acct_db.* to 'debian'@'kapoorlabs.fr';
flush privileges;
exit
```



### Download, build, and install Slurm
Download tar.bz2 from https://www.schedmd.com/downloads.php 

```console
$ 
$ wget https://download.schedmd.com/slurm/slurm-23.02.0.tar.bz2
$ tar xvjf slurm-23.02.0.tar.bz2
$ cd slurm-23.02.0
$ ./configure --with-hdf5=no
$ make
$ make install

Create a slurm configuration file with https://slurm.schedmd.com/configurator.html
```


## Open a port
```console
$ check if port is being used: netstat -na | grep:6819
$ sudo ufw allow 6819/firewall-cmd --add-port=6819/tcp
$ Test if port is open: ls | nc -l -p 6819
```






