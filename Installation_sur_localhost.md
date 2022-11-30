# Tentative d'installation de Slurm en local

On tente l'installation à la main sur un poste local (Ubuntu 22.04.1 LTS)

On a la doc d'installation officielle : [Quick Start Administrator Guide](https://slurm.schedmd.com/quickstart_admin.html)

Et une vidéo [Youtube](https://www.youtube.com/watch?v=VozfZGZIX8w)

Et il existe un rôle Ansible permettant l'installation : [Ansible-slurm](https://github.com/galaxyproject/ansible-slurm)

On suit ce [tuto](https://github.com/nateGeorge/slurm_gpu_ubuntu)

# Installation de Munge

Guide d'installation : [Installation Guide](https://github.com/dun/munge/wiki/Installation-Guide)

## Ajout des users et groupes

> sudo adduser -u 1111 munge --disabled-password --gecos ""

> sudo adduser -u 1121 slurm --disabled-password --gecos ""

## Installation du service Munge

> sudo apt-get install libmunge-dev libmunge2 munge -y

> sudo systemctl enable munge

> sudo systemctl start munge

Test

> munge -n | unmunge | grep STATUS

En local pas de recopie de clé, mais on vérifie qu'elle existe

> sudo less /etc/munge/munge.key

# Installation d'une base de données pour l'accounting de Slurm

Le tuto propose d'installer MariaDB, tente d'utiliser PostgreSQL qui est déjà présent ... mais ce n'est pas possible, on revient sur MariaDB

On dispose déjà d'une instance de MariaDB, v15.1

> sudo mysql -u root

	create database slurm_acct_db;
	
	create user 'slurm'@'localhost';
	
	set password for 'slurm'@'localhost' = password('slurmdbpass');
	
	grant usage on *.* to 'slurm'@'localhost';
	
	grant all privileges on slurm_acct_db.* to 'slurm'@'localhost';
	
	flush privileges;
	
	exit



    
    
# Installation de Slurm

## Construction de l'image 

> cd workspace

> mkdir slurm

> cd slurm

> wget https://download.schedmd.com/slurm/slurm-22.05.6.tar.bz2

> tar xvjf slurm-22.05.6.tar.bz2

> cd slurm-22.05.6

> ./configure --prefix=/tmp/slurm-build --sysconfdir=/etc/slurm --enable-pam --with-pam_dir=/lib/x86_64-linux-gnu/security/ --without-shared-libslurm

> make

Ca plante : home/ign.fr/BPesty/eclipse-workspace/slurm/slurm-22.05.6/src/plugins/acct_gather_profile/hdf5/sh5util/sh5util.c:1078: undefined reference to `H5PTopen'

On  installe la librairie manquante :

> sudo apt-get install libhdf5-dev

Et on relance

> make

Pas mieux, on reconfigure en désactivant hdf5

> sudo ./configure --prefix=/tmp/slurm-build --sysconfdir=/etc/slurm --enable-pam --with-pam_dir=/lib/x86_64-linux-gnu/security/ --without-shared-libslurm --with-hdf5=no

Et on relance

> sudo make

> sudo make contrib

> sudo make install

> cd ..

## Installation

Il nous faut fpm ...

> sudo apt-get install ruby ruby-dev rubygems build-essential

> sudo gem install fpm

Test

> fpm -v

On continue  

> sudo fpm -s dir -t deb -v 1.0 -n slurm-22.05.6 --prefix=/usr -C /tmp/slurm-build .

> sudo dpkg -i slurm-22.05.6_1.0_amd64.deb

## Création des répertoires de config

> sudo mkdir -p /etc/slurm /etc/slurm/prolog.d /etc/slurm/epilog.d /var/spool/slurm/ctld /var/spool/slurm/d /var/log/slurm

> sudo chown slurm /var/spool/slurm/ctld /var/spool/slurm/d /var/log/slurm

> sudo cp ./slurm-22.05.6/etc/slurmdbd.service /etc/systemd/system/

> sudo cp ./slurm-22.05.6/etc/slurmctld.service /etc/systemd/system/

On jette un oeil aux fichiers :

> sudo nano /etc/systemd/system/slurmdbd.service

	[Unit]
	Description=Slurm DBD accounting daemon
	After=network-online.target munge.service postgresql.service
	Wants=network-online.target
	ConditionPathExists=/etc/slurm/slurmdbd.conf
	
	[Service]
	Type=simple
	EnvironmentFile=-/etc/sysconfig/slurmdbd
	EnvironmentFile=-/etc/default/slurmdbd
	ExecStart=/tmp/slurm-build/sbin/slurmdbd -D -s $SLURMDBD_OPTIONS
	ExecReload=/bin/kill -HUP $MAINPID
	LimitNOFILE=65536
	TasksMax=infinity
	
	# Uncomment the following lines to disable logging through journald.
	# NOTE: It may be preferable to set these through an override file instead.
	#StandardOutput=null
	#StandardError=null
	
	[Install]
	WantedBy=multi-user.target

> sudo nano /etc/systemd/system/slurmctld.service

	[Unit]
	Description=Slurm controller daemon
	After=network-online.target munge.service
	Wants=network-online.target
	ConditionPathExists=/etc/slurm/slurm.conf
	
	[Service]
	Type=simple
	EnvironmentFile=-/etc/sysconfig/slurmctld
	EnvironmentFile=-/etc/default/slurmctld
	ExecStart=/tmp/slurm-build/sbin/slurmctld -D -s $SLURMCTLD_OPTIONS
	ExecReload=/bin/kill -HUP $MAINPID
	LimitNOFILE=65536
	TasksMax=infinity
	
	# Uncomment the following lines to disable logging through journald.
	# NOTE: It may be preferable to set these through an override file instead.
	#StandardOutput=null
	#StandardError=null
	
	[Install]
	WantedBy=multi-user.target

On recopie le fichier de config pour la base de données :

> sudo cp ./slurm-22.05.6/etc/slurmdbd.conf.example /etc/slurm/slurmdbd.conf

On l'édite

> sudo nano /etc/slurm/slurmdbd.conf

	#
	# slurmdbd.conf file.
	#
	# See the slurmdbd.conf man page for more information.
	#
	# Archive info
	#ArchiveJobs=yes
	#ArchiveDir="/tmp"
	#ArchiveSteps=yes
	#ArchiveScript=
	#JobPurge=12
	#StepPurge=1
	#
	# Authentication info
	AuthType=auth/munge
	#AuthInfo=/var/run/munge/munge.socket.2
	#
	# slurmDBD info
	DbdAddr=localhost
	DbdHost=localhost
	#DbdPort=7031
	SlurmUser=slurm
	#MessageTimeout=300
	DebugLevel=verbose
	#DefaultQOS=normal,standby
	LogFile=/var/log/slurm/slurmdbd.log
	PidFile=/var/run/slurmdbd.pid
	#PluginDir=/usr/lib/slurm
	#PrivateData=accounts,users,usage,jobs
	#TrackWCKey=yes
	#
	# Database info
	StorageType=accounting_storage/mysql
	#StorageHost=localhost 
	StoragePort=1234
	StoragePass=slurmdbpass
	StorageUser=slurm
	#StorageLoc=slurm_acct_db


## Démarrage des services


> sudo systemctl daemon-reload

> sudo systemctl enable slurmdbd

> sudo systemctl start slurmdbd

> sudo systemctl enable slurmctld

> sudo systemctl start slurmctld


On vérifie :

> sudo service slurmdbd status

	fatal: slurmdbd.conf file /etc/slurm/slurmdbd.conf should be 600 is 644 accessible for group or others
	
On corrige les droits

> sudo chmod 600 /etc/slurm/slurmdbd.conf

> sudo chown -R slurm:slurm /etc/slurm

> sudo service slurmdbd restart

> sudo service slurmdbd status

Ca démarre mais avec des warnings

	nov. 24 17:09:39 HP2011P080 systemd[1]: Started Slurm DBD accounting daemon.
	nov. 24 17:09:39 HP2011P080 slurmdbd[253891]: slurmdbd: pidfile not locked, assuming no running daemon
	nov. 24 17:09:40 HP2011P080 slurmdbd[253891]: slurmdbd: accounting_storage/as_mysql: _check_mysql_concat_is_sane: MySQL server version is: 5.5.5-10.6.11-MariaDB-0ubuntu0.22.04.1
	nov. 24 17:09:40 HP2011P080 slurmdbd[253891]: slurmdbd: error: Database settings not recommended values: innodb_buffer_pool_size innodb_lock_wait_timeout
	nov. 24 17:09:40 HP2011P080 slurmdbd[253891]: slurmdbd: accounting_storage/as_mysql: init: Accounting storage MYSQL plugin loaded
	
On met à jour la config

> sudo nano /etc/mysql/my.cnf

	[mysqld]
	innodb_buffer_pool_size=4096M
	innodb_log_file_size=64M
	innodb_lock_wait_timeout=900

> sudo service mysql restart

> sudo service slurmdbd restart

> sudo service slurmdbd status

Cette fois ça a l'air bon

> sudo service slurmctld restart

> sudo service slurmctld status

On recopie le fichier de config pour surl :

> sudo cp ./slurm-22.05.6/etc/slurm.conf.example /etc/slurm/slurm.conf

On va sur le [configurator](https://slurm.schedmd.com/configurator.html) pour générer un fichier de config

> sudo nano /etc/slurm/slurm.conf

On redémarre :

	nov. 24 16:20:49 HP2011P080 slurmctld[249773]: error: Parse error in file /etc/slurm/slurm.conf line 21: " KillWait=30"
	nov. 24 16:20:49 HP2011P080 slurmctld[249773]: error: Parse error in file /etc/slurm/slurm.conf line 25: " SlurmdTimeout=300"
	nov. 24 16:20:49 HP2011P080 slurmctld[249773]: error: Parse error in file /etc/slurm/slurm.conf line 31: " SelectType=select/cons_tres"
	nov. 24 16:20:49 HP2011P080 slurmctld[249773]: error: Parse error in file /etc/slurm/slurm.conf line 43: " JobAcctGatherType=jobacct_gather/none SlurmctldDebug=info SlurmctldLogFile=/var/log/slurmctld.lo>
	nov. 24 16:20:49 HP2011P080 systemd[1]: slurmctld.service: Failed with result 'exit-code'.
	nov. 24 16:20:49 HP2011P080 slurmctld[249773]: error: No SlurmctldHost defined.
	nov. 24 16:20:49 HP2011P080 slurmctld[249773]: fatal: Unable to process configuration file

Il manque des retour chariot dans le fichier généré par le configurator..., on corrige manuellement.

On redémarre, ça plante toujours.

On regarde les logs qui sont normalement dans /var/log/slurmd.log et /var/log/slurmctld.log

	fatal: mkdir(/var/spool/slurmctld): Permission denied
	
On crée le répertoire et on retente

> sudo mkdir -p /var/spool/slurmctld

> sudo chown slurm /var/spool/slurmctld

Le service slurm a finalement l'air d'avoir bien démarré

	● slurmctld.service - Slurm controller daemon
	     Loaded: loaded (/etc/systemd/system/slurmctld.service; enabled; vendor preset: enabled)
	     Active: active (running) since Thu 2022-11-24 16:29:57 CET; 3s ago
	   Main PID: 251282 (slurmctld)
	      Tasks: 10
	     Memory: 3.7M
	        CPU: 27ms
	     CGroup: /system.slice/slurmctld.service
	             ├─251282 /tmp/slurm-build/sbin/slurmctld -D -s
	             └─251286 "slurmctld: slurmscriptd" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
	
	nov. 24 16:29:57 HP2011P080 systemd[1]: Started Slurm controller daemon.
	nov. 24 16:29:57 HP2011P080 slurmctld[251282]: slurmctld: No parameter for mcs plugin, default values set
	nov. 24 16:29:57 HP2011P080 slurmctld[251282]: slurmctld: mcs: MCSParameters = (null). ondemand set.


## Démarrage d'un worker

A faire sur chaque machine du cluster en théorie

> sudo dpkg -i slurm-22.05.6_amd64.deb

> sudo cp ./slurm-22.05.6/etc/slurmd.service /etc/systemd/system/

> sudo systemctl enable slurmd

> sudo systemctl start slurmd

> sudo systemctl status slurmd

Failed

	× slurmd.service - Slurm node daemon
	     Loaded: loaded (/etc/systemd/system/slurmd.service; enabled; vendor preset: enabled)
	     Active: failed (Result: exit-code) since Thu 2022-11-24 17:19:12 CET; 12s ago
	    Process: 255405 ExecStart=/tmp/slurm-build/sbin/slurmd -D -s $SLURMD_OPTIONS (code=exited, status=1/FAILURE)
	   Main PID: 255405 (code=exited, status=1/FAILURE)
	        CPU: 19ms
	
	nov. 24 17:19:12 HP2011P080 systemd[1]: Started Slurm node daemon.
	nov. 24 17:19:12 HP2011P080 systemd[1]: slurmd.service: Main process exited, code=exited, status=1/FAILURE
	nov. 24 17:19:12 HP2011P080 systemd[1]: slurmd.service: Failed with result 'exit-code'.

On vérifie les logs

> sudo tail -n 500 /var/log/slurmd.log

	[2022-11-24T17:19:12.589] error: Domain socket directory /var/spool/slurmd: No such file or directory
	[2022-11-24T17:19:12.607] error: Node configuration differs from hardware: CPUs=12:12(hw) Boards=1:1(hw) SocketsPerBoard=12:1(hw) CoresPerSocket=1:6(hw) ThreadsPerCore=1:2(hw)
	[2022-11-24T17:19:12.607] error: Couldn't find the specified plugin name for cgroup/v2 looking at all files
	[2022-11-24T17:19:12.609] error: cannot find cgroup plugin for cgroup/v2
	[2022-11-24T17:19:12.609] error: cannot create cgroup context for cgroup/v2
	[2022-11-24T17:19:12.609] error: Unable to initialize cgroup plugin
	[2022-11-24T17:19:12.609] error: slurmd initialization failed

On créé le répertoire manquant

> sudo mkdir -p /var/spool/slurmd

> sudo chown slurm /var/spool/slurmd

On édite la config

**Note :**ce qui est intéressant selon [ce site](https://rolk.github.io/2015/04/20/slurm-cluster) c'est de récupérer les infos directement sur la machine.

La commande lscpu donne des infos.

	NodeName: echo $(hostname -s) 
	CPUs: nproc --all
	RealMemory: echo $(grep "^MemTotal:" /proc/meminfo | awk '{print int($2/1024)}') 
	Sockets : echo $(grep "^physical id" /proc/cpuinfo | sort -uf | wc -l) 
	ThreadsPerCore: lscpu | grep -E '^Thread|^Core|^Socket|^CPU\('
	CoresPerSocket: echo $(grep "^siblings" /proc/cpuinfo | head -n 1 | awk '{print $3}') à diviser par ThreadsPerCore
	State=UNKNOWN


> sudo nano /etc/slurm/slurm.conf


On redémarre

> sudo systemctl restart slurmd

> sudo systemctl status slurmd

Il reste des erreurs	

> sudo tail -n 500 /var/log/slurmd.log
	
[2022-11-24T17:40:07.952] error: Couldn't find the specified plugin name for cgroup/v2 looking at all files
[2022-11-24T17:40:07.955] error: cannot find cgroup plugin for cgroup/v2
[2022-11-24T17:40:07.955] error: cannot create cgroup context for cgroup/v2
[2022-11-24T17:40:07.955] error: Unable to initialize cgroup plugin
[2022-11-24T17:40:07.955] error: slurmd initialization failed

On désactive le plugin cgroup

> sudo nano /etc/slurm/slurm.conf

	ProctrackType=proctrack/linuxproc
	AccountingStorageType=accounting_storage/slurmdbd
	
On redémarre

> sudo systemctl restart slurmd

> sudo systemctl status slurmd

Toujours pas, il nous manque le fichier de config des cgroup

> sudo cp ./slurm-22.05.6/etc/cgroup.conf.example /etc/slurm/cgroup.conf

> sudo nano /etc/slurm/cgroup.conf

> sudo chown -R slurm:slurm /etc/slurm 

Toujours pas, une piste trouvée [ici](https://www.reddit.com/r/SLURM/comments/vjquih/error_cannot_find_cgroup_plugin_for_cgroupv2/)

	Based on slurm-22.05.4.tar.bz2, it looks like the tar boll file missing some source codes for cgroup/v2 plugin. 
	So I solved this problem by adding this line on /etc/slurm/cgroup.conf CgroupPlugin=cgroup/v1

> sudo nano /etc/slurm/cgroup.conf

	CgroupPlugin=cgroup/v1

Finalement ça démarre

	● slurmd.service - Slurm node daemon
	     Loaded: loaded (/etc/systemd/system/slurmd.service; enabled; vendor preset: enabled)
	     Active: active (running) since Thu 2022-11-24 18:04:35 CET; 3s ago
	   Main PID: 262383 (slurmd)
	      Tasks: 1
	     Memory: 2.1M
	        CPU: 34ms
	     CGroup: /system.slice/slurmd.service
	             └─262383 /tmp/slurm-build/sbin/slurmd -D -s


# Tests

La commande "sinfo" devrait afficher des infos sur le cluster, mais ça plante.

	sinfo: error: PluginDir: /tmp/slurm-build/lib/slurm: No such file or directory
	sinfo: error: Bad value "/tmp/slurm-build/lib/slurm" for PluginDir
	sinfo: fatal: Unable to process configuration file

En fait après vérification, tout le service slurm a été installé dans /tmp, et a été purgé.

Les fichiers de config des services pointent aussi sur le /tmp.

On recommence la procédure d'installation à partir de l'étape configure.

> cd ~/eclipse-workspace/slurm/slurm-22.05.6/

> sudo make clean

> sudo ./configure --prefix=/usr --sysconfdir=/etc/slurm --enable-pam --with-pam_dir=/lib/x86_64-linux-gnu/security/ --without-shared-libslurm --with-hdf5=no

> sudo make

> sudo make contrib

> sudo make install

> cd ..

> sudo fpm -s dir -t deb -v 1.0 -n slurm-22.05.6 --prefix=/usr .

> sudo dpkg -i slurm-22.05.6_1.0_amd64.deb

A priori c'est bon, les binaires sont installés dans /usr/sbin/

On réédite les fichers de config des services pour mettre à jour ce chemin

> sudo nano /etc/systemd/system/slurmctld.service

> sudo nano /etc/systemd/system/slurmdbd.service

> sudo nano /etc/systemd/system/slurmd.service

> sudo systemctl daemon-reload

Et on redémarre les services

> sudo service slurmctld restart

> sudo service slurmdbd restart

> sudo service slurmd restart

Et c'est enfin bon

# TODO : Configuration des ressources avec GRES ?


> sudo nano /etc/slurm/gres.conf

	##################################################################
	# Slurm's Generic Resource (GRES) configuration file
	# Define GPU devices with MPS support, with AutoDetect sanity checking
	##################################################################
	AutoDetect=nvml
	Name=gpu File=/dev/nvidia0 COREs=0


# Tests

cf [Commandes](Commandes.md)



	