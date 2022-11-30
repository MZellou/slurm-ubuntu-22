# Slurm - Commandes usuelles

Cf aussi [slurm](https://mesocentre.univ-amu.fr/slurm/)

## sinfo - view or modify Slurm configuration and state

Affiche des infos sur les partitions

> sinfo 

	PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
	debug*       up   infinite      1   idle HP2011P080

On peut aussi afficher des infos sur les machines avec :

> sudo slurmd -C

	NodeName=HP2011P080 CPUs=12 Boards=1 SocketsPerBoard=1 CoresPerSocket=6 ThreadsPerCore=2 RealMemory=15807
	UpTime=0-06:03:05
	
## scontrol - view or modify Slurm configuration and state

Permet de tester si les controleurs du cluster répondent

> scontrol ping

## srun - Run parallel jobs

Soumission d'un job en intéractif

On peut lancer la commande "hostname" en 8 exemplaires

> srun -n 8 hostname

## squeue - View information about jobs located in the Slurm scheduling queue

Affiche la list des jobs en attente sur une partition

> squeue

	             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
	
	
	
## sbatch - Submit a batch script to Slurm.


Lance un batch (script shell) en interprétant les commantaires #BATCH 

> sbatch submit_job.sh

Rend la main instantanément
