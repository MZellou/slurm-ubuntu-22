# Installation of SLURM 22 on Ubuntu 22

Credits :

* <https://slurm.schedmd.com>
  
* <https://docs.alliancecan.ca>

* <https://bugs.schedmd.com>

* <https://hpc-uit.readthedocs.io>

## Dist upgrade on all nodes (master and workers)

Google it

## Synchronizing time (source : <https://knowm.org/how-to-synchronize-time-across-a-linux-cluster/>)

Clocks needs to be synchronized for all the nodes. As far as I understand, it creates a local time for master and sync workers to that local clock.

### On master node

```bash
sudo apt-get install ntp
sudo dpkg-reconfigure tzdata
```

```bash
sudo nano /etc/ntp.conf
```

Add the following to provide your current local time as a default should you temporarily (or permanently) lose Internet connectivity:

```bash
server 127.127.1.0
fudge 127.127.1.0 stratum 10 
```

```bash
sudo /etc/init.d/ntp restart
```

## On workers

```bash
sudo apt-get install ntp
sudo nano /etc/ntp.conf
```

Add at the end of the file :

```bash
server <main time server ip> iburst
```

Remove the following

```bash
# Use servers from the NTP Pool Project. Approved by Ubuntu Technical Board
# on 2011-02-08 (LP: #104525). See http://www.pool.ntp.org/join.html for
# more information.
#server 0.ubuntu.pool.ntp.org
#server 1.ubuntu.pool.ntp.org
#server 2.ubuntu.pool.ntp.org
#server 3.ubuntu.pool.ntp.org
 
# Use Ubuntu's ntp server as a fallback.
#server ntp.ubuntu.com
 ```

Restart on all nodes

```bash
sudo /etc/init.d/ntp restart
```

Check connectivity :

```bash
ntpq -c lpeer
```

## Set up munge and slurm users and groups

The following groups and users should be synchronized between all nodes with the same UID/GID for all nodes.

If users already exists with the wrong UID/GID :

```bash
sudo deluser munge
sudo deluser slurm
```

```bash
sudo addgroup -gid 111111 munge
sudo addgroup -gid 111121 slurm
sudo adduser -u 111111 munge --disabled-password --gecos "" -gid 111111
sudo adduser -u 111121 slurm --disabled-password --gecos "" -gid 111121
```

Remove /home/munge and /home/slurm if necessary

## Parallel ssh (TODO)

sudo pip install parallel-ssh
nano hosts

## On workers with GPU, update nvidia-drivers (source : <https://linuxconfig.org/how-to-install-the-nvidia-drivers-on-ubuntu-20-04-focal-fossa-linux>)

### Search for latests

```bash
sudo apt-cache search nvidia-driver*
```

### Install

```bash
sudo aptitude install nvidia-driver-520
```

Reboot

## Install NFS (shared storage / TODO ?)

```bash
sudo mkdir /media/hd/slurm-storage
```

```bash
sudo chown dlsupport:dlsupport /media/hd/slurm-storage
```

Next we need to add rules for the shared location. This is done with:

```bash
sudo nano /etc/exports
```

ssh-copy-id mzellouadmin@DEL1910W010.ign.fr
Then adding the line:

```bash
/media/hd/slurm-storage 172.20.252.19(rw,sync,no_root_squash,all_squash,anonuid=999999,anongid=999999) 172.20.0.61(rw,sync,no_root_squash,all_squash,anonuid=999999,anongid=999999)
```

```bash
sudo systemctl start nfs-kernel-server.service
```

```bash
sudo ufw allow from 172.20.0.61 to any port nfs
```

uwf inactive ???
TODO use store-dai

## Passwordless SSH from master to all workers

From/To workers to master ??

```bash
ssh-copy-id admin@worker
ssh-copy-id worker@admin
```

Mount with your own id // security issue (slurm & munge should have rights to store-dai ??)

```bash
sudo mount -v -t cifs -o rw,user=xxx,domain=IGN,uid=xxx,gid=xxx,forceuid,forcegid,file_mode=0777,dir_mode=0777 //store.ign.fr/store-echange /media/store-echange
```

## Install munge on the master

Good starting points :

* <https://github.com/mknoxnv/ubuntu-slurm.git>
  
* <https://github.com/nateGeorge/slurm_gpu_ubuntu>

```bash
sudo apt-get install libmunge-dev libmunge2 munge -y
sudo systemctl enable munge
sudo systemctl start munge
```

Test munge if you like:

```bash
munge -n | unmunge | grep STATUS
```

Create munge key

```bash
sudo -u munge /usr/sbin/mungekey --verbose
```

Copy the munge key to /media/store-echange/slurm/storage

```bash
sudo cp /etc/munge/munge.key /media/store-echange/slurm/storage/

```

## install munge on workers

```bash
sudo apt-get install libmunge-dev libmunge2 munge

sudo cp /media/store-echange/slurm/storage/munge.key /etc/munge/munge.key
sudo chown -R munge: /etc/munge
sudo chmod 0600 /etc/munge/munge.key

sudo chown -R munge:munge /var/log/munge
sudo chmod 0700 /var/log/munge/munged.log

sudo systemctl enable munge.service
sudo systemctl start munge.service
```

## Install DB components on master

```bash
sudo apt-get install wget software-properties-common dirmngr ca-certificates apt-transport-https -y
sudo apt-get install git gcc make ruby ruby-dev libpam0g-dev mysql-client mysql-server build-essential libssl-dev -y
sudo apt-get install mariadb-client mariadb-server libmariadb-dev-compat -y
sudo gem install fpm
```

```bash
sudo chown -R mysql:mysql /var/lib/mysql
sudo systemctl enable mysql
sudo systemctl start mysql
sudo mysql -u root
```

Within mysql create db with specific user/pass. Should be the same on slurm.conf:

```bash
create database slurm_acct_db;
create user 'slurm'@'localhost';
set password for 'slurm'@'localhost' = password('slurmdbpass');
grant usage on *.* to 'slurm'@'localhost';
grant all privileges on slurm_acct_db.* to 'slurm'@'localhost';
flush privileges;
exit
```

Ideally you want to change the password to something different than slurmdbpass.
This must also be set in the config file /storage/slurmdbd.conf

## Install slurm on master

Build SLURM from sources.
TODO fix libpaths and prefix to local home folders.

```bash
cd /media/store-echange/slurm/storage
wget https://download.schedmd.com/slurm/slurm-22.05.6.tar.bz2
tar xvjf slurm-22.05.6.tar.bz2
cp -r /media/store-echange/slurm/storage/slurm-22.05.6 ~/slurm/install_dir/slurm-22.05.6
cd slurm-22.05.6
cd ~/slurm/install_dir/slurm-22.05.6
./configure --help
sudo ./configure --enable-debug --prefix=/tmp/slurm-build --sysconfdir=/etc/slurm --with-mysql_config=/usr/bin  --libdir=/usr/lib
make
make contrib
make install
cd ..

sudo fpm -s dir -t deb -v 1.0 -n slurm-22.05.6 --prefix=/usr -C /tmp/slurm-build .
sudo dpkg -i slurm-22.05.6_1.0_amd64.deb
```

### Master logs and run folders

```bash
sudo mkdir -p /var/spool/slurmctld /etc/slurm /var/spool/slurmd /var/log/slurm /var/lib/slurm /var/lib/slurm/slurmd /var/lib/slurm/slurmctld /run/slurm
sudo chown -R slurm:slurm /var/spool/slurmctld /etc/slurm /var/spool/slurmd /var/log/slurm /var/lib/slurm /var/lib/slurm/slurmd /var/lib/slurm/slurmctld /run/slurm /var/spool/slurm

sudo chmod 755 /var/spool/slurmctld

sudo chown slurm:slurm /var/log/slurm/slurmctld.log
sudo chown slurm:slurm /var/log/slurm/slurmdbd.log
```

Copy SLURM confs
TODO use systemctl

```bash
sudo cp /media/store-echange/slurm/storage/conf/slurm-22/slurmdbd.service /etc/systemd/system/
sudo cp /media/store-echange/slurm/storage/conf/slurm-22/slurmctld.service /etc/systemd/system/

sudo cp /media/store-echange/slurm/storage/conf/slurm-22/slurmdbd.conf /etc/slurm/

sudo chown -R slurm:slurm /etc/systemd/system/slurmdbd.service
sudo chown -R slurm:slurm /etc/systemd/system/slurmctld.service

sudo chmod 600 /etc/slurm/slurmdbd.conf
sudo chown slurm: /etc/slurm/slurmdbd.conf

sudo cp /media/store-echange/slurm/storage/conf/slurm-22/cgroup* /etc/slurm/
sudo cp /media/store-echange/slurm/storage/conf/slurm-22/slurm.conf /etc/slurm/
sudo cp /media/store-echange/slurm/storage/conf/slurm-22/gres.conf /etc/slurm/

sudo systemctl daemon-reload
sudo systemctl enable slurmdbd
sudo systemctl start slurmdbd
sudo systemctl enable slurmctld
sudo systemctl start slurmctld

```

Check status

```bash
sudo systemctl status slurmdbd
journalctl -u slurmdbd
sudo systemctl restart slurmdbd

sudo systemctl status slurmctld
journalctl -u slurmctld
sudo systemctl restart slurmctld
```

(What I use right now) Run DBD and CTLD direcly :

```bash
sudo systemctl daemon-reload
sudo /usr/sbin/slurmdbd -Dvvvv
sudo /usr/sbin/slurmctld -Dvvvv
```

If the master is also going to be a worker/compute node, you should do:

```bash
sudo cp /media/store-echange/slurm/storage/conf/slurm-22/slurmd.service /etc/systemd/system/
sudo systemctl enable slurmd
sudo systemctl start slurmd
```

## Install slurm on workers

TODO should work only unpackaging the deb file

```bash

cd /media/store-echange/slurm/storage
wget https://download.schedmd.com/slurm/slurm-22.05.6.tar.bz2
tar xvjf slurm-22.05.6.tar.bz2
cp -r /media/store-echange/slurm/storage/slurm-22.05.6 ~/slurm/install_dir/slurm-22.05.6
cd slurm-22.05.6
cd ~/slurm/install_dir/slurm-22.05.6
./configure --help
sudo ./configure --enable-debug --prefix=/tmp/slurm-build --sysconfdir=/etc/slurm --libdir=/usr/lib
make
make contrib
make install
cd ..

sudo fpm -s dir -t deb -v 1.0 -n slurm-22.05.6 --prefix=/usr -C /tmp/slurm-build .
sudo dpkg -i slurm-22.05.6_1.0_amd64.deb
```

```bash
cd /media/store-echange/slurm/storage
sudo dpkg -i /media/store-echange/slurm/storage/slurm-22.05.6_1.0_amd64.deb
```

Copy confs. Should be the same on master and workers.

```bash
sudo cp /media/store-echange/slurm/storage/conf/slurm-22/slurmd.service /etc/systemd/system/

sudo mkdir -p /etc/slurm /var/spool/slurmd /var/log/slurm /var/lib/slurm /var/lib/slurm/slurmd /run/slurm
sudo chown -R slurm:slurm /etc/slurm /var/spool/slurmd /var/log/slurm /var/lib/slurm /var/lib/slurm/slurmd /run/slurm
```

TODO : fix lib path in compilation config

```bash
sudo cp -r /media/store-echange/slurm/storage/slurm /usr/lib64/
```

```bash
sudo cp /media/store-echange/slurm/storage/conf/slurm-22/cgroup* /etc/slurm/
sudo cp /media/store-echange/slurm/storage/conf/slurm-22/slurm.conf /etc/slurm/
sudo cp /media/store-echange/slurm/storage/conf/slurm-22/gres.conf /etc/slurm/

sudo systemctl daemon-reload
sudo systemctl enable slurmd
sudo systemctl start slurmd
```

Or use

```bash
sudo /usr/sbin/slurmd -Dvvvv
```

### Troubleshooting helpers

Shared library

```bash
ldd /usr/sbin/slurmdbd
```

```bash
sudo ldconfig
```

Troubleshoot munge

```bash
echo foo | munge | unmunge

echo foo | ssh DEL1910W010.ign.fr munge | unmunge
```
