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
Test if all is running well

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
$ ./configure --prefix=/tmp/slurm-build --sysconfdir=/etc/debian --enable-pam --with-pam_dir=/lib/x86_64-linux-gnu/security/ --without-shared-libslurm --with-hdf5=no
$ make
$ make contrib
$ make install
$ cd ..
$ fpm -s dir -t deb -v 1.0 -n slurm-23.02.0 --prefix=/usr -C /tmp/slurm-build .
$ dpkg -i slurm-23.02.0_1.0_amd64.deb
$ 
$ mkdir -p /etc/debian /etc/debian/prolog.d /etc/debian/epilog.d /var/spool/debian/ctld /var/spool/debian/d /var/log/debian
$ chown debian /var/spool/debian/ctld /var/spool/debian/d /var/log/debian
```


```console
Copy into place config files from this repo which you've already cloned into 
$ 
$ cp slurmdbd.service /etc/systemd/system/
$ cp slurmctld.service /etc/systemd/system/
```




```console
$ systemctl daemon-reload
$ systemctl enable slurmdbd
$ systemctl start slurmdbd
$ systemctl enable slurmctld
$ systemctl start slurmctld
$ systemctl start slurmdbd
```

## Open a port
```console
$ check if port is being used: netstat -na | grep:6819
$ sudo ufw allow 6819/firewall-cmd --add-port=6819/tcp
$ Test if port is open: ls | nc -l -p 6819
```


## Create initial slurm cluster, account, and user.
```console
$ sacctmgr add cluster compute-cluster
$ sacctmgr add account compute-account description="Compute accounts" Organization=OurOrg
$ sacctmgr create user myuser account=compute-account adminlevel=None
$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
debug*       up   infinite      0    n/a
```

## Install slurm and associated components on a compute node.

### Install munge
MUNGE (MUNGE Uid 'N' Gid Emporium) is an authentication service for creating and validating credentials.
https://dun.github.io/munge/
```console
$ apt-get update
$ apt-get install libmunge-dev libmunge2 munge
$ scp slurm-ctrl:/etc/munge/munge.key /etc/munge/
$ chown munge:munge /etc/munge/munge.key
$ chmod 400 /etc/munge/munge.key
```

Ubuntu 16.04
```console
$ systemctl enable munge
$ systemctl restart munge
```



### Test munge
```console
$ munge -n | unmunge | grep STATUS
STATUS:           Success (0)
$ munge -n | ssh slurm-ctrl unmunge | grep STATUS
STATUS:           Success (0)
```

### Install Slurm
```console
$ dpkg -i /storage/slurm-17.02.6_1.0_amd64.deb
$ mkdir /etc/slurm
$ cp /storage/ubuntu-slurm/slurm.conf /etc/slurm/slurm.conf

If necessary modify gres.conf to reflect the properties of this compute node.
gres.conf.dgx is an example configuration for the DGX-1. 
Use "nvidia-smi topo -m" to find the GPU-CPU affinity.

The node-config.sh script will, if run on the compute node, output the appropriate lines to
add to slurm.conf and gres.conf.

$ cp /storage/ubuntu-slurm/gres.conf /etc/slurm/gres.conf
$ cp /storage/ubuntu-slurm/cgroup.conf /etc/slurm/cgroup.conf
$ cp /storage/ubuntu-slurm/cgroup_allowed_devices_file.conf /etc/slurm/cgroup_allowed_devices_file.conf
$ useradd slurm
$ mkdir -p /var/spool/slurm/d
```


```console
$ cp /storage/ubuntu-slurm/slurmd.service /etc/systemd/system/
$ systemctl enable slurmd
$ systemctl start slurmd
```



## Test Slurm
```console
$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
debug*       up   infinite      1   idle linux1
```

### Set up cgroups
Using memory cgroups to restrict jobs to allocated memory resources requires setting kernel parameters
```console
$ vi /etc/default/grub
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
$ update-grub
$ reboot
```

## Run a job from slurm-ctrl
```console
$ su - myuser
$ srun -N 1 hostname
linux1
```
## Run a GPU job from slurm-ctrl
```console
$ srun -N 1 --gres=gpu:1 env | grep CUDA
CUDA_VISIBLE_DEVICES=0
```

## Enable Slurm PAM SSH Control
This prevents users from ssh-ing into a compute node on which they do not have an allocation.

On the compute nodes:
```console
$ cp /storage/slurm-17.02.6/contribs/pam/.libs/pam_slurm.so /lib/x86_64-linux-gnu/security/
$ vi /etc/pam.d/sshd
account    required     /lib/x86_64-linux-gnu/security/pam_slurm.so
```

If you are using something such as LDAP for user accounts and want to allow local system 
accounts (for example, a non-root local admin account) to login without going through 
slurm make the following change.  Add this line to the beginning of the sshd file.

```console
$ vi /etc/pam.d/sshd
account    sufficient   pam_localuser.so
```

On slurm-ctrl as non-root user
```console
$ ssh linux1 hostname
Access denied: user myuser (uid=1000) has no active jobs on this node.
Connection to linux1 closed by remote host.
Connection to linux1 closed.
$ salloc -N 1 -w linux1
$ ssh linux1 hostname
linux1
```





