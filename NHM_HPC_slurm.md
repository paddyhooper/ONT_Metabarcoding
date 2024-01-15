# Using the NHM HPC Server and slurm

# What is the HPC cluster?

The NHM's HPC (High Performance Computing) cluster is a group of servers on which you can run scientific analyses and computations.

# Structure of the HPC

![Diagram](https://wpweb-prod.rtu.lv/hpc/wp-content/uploads/sites/77/2016/07/Kas-ir-HPC-495x400.png)  

## Head node 

The head node is hpc-head-002 and is the node you will normally log into.    
The head node is mainly used to explore the file hierarchy, move/transfer files and folders, install software, and submit analyses through the job scheduler (Slurm). 

## Compute nodes

The compute nodes are where your jobs are processed, and are not accessible directly through SSH

There are three compute nodes:  

1. hpc-cpu-001 - compute node with 96 CPUs (192 threads) and 2 TB ram  
2. hpc-gpu-001 - compute node with GPU, 96 CPUs (192 threads) and NVIDIA Tesla V100S 32GB GPU
3. hpc-gpu-002 - compute node with GPU, 96 CPUS (192 threads) and NVIDIA Tesla V100S 32GB GPU

**You can ONLY SSH into the Head Node and access the CPU nodes from the Head Node

# Saving data on the NHM HPC Servers

## Personal workspace A.K.A Home Directory

> When you log in you will be in your home directory, which is a personal area that only you have access to. You can save files and install software but the space is limited to 50GB.
>
>Therefore, we save important files in our group workspaces

## Group workspaces

> Your work should ideally be kept in a group workspace. Group workspaces are located in /workspaces/groups and are organized by project. If you need a new group folder for a project you are working on, contact TS to request one. Access can also be granted to your colleagues, so group workspaces are ideal for collaboration. The default size limit is 2TB, but TS can increase this if necessary.
>
> You can navigate to a group workspace via any of the following symlinks:
>/gpfs/nhmfsa/bulk/share/data/mbl/share/workspaces/groups/<folder_name>

## Scratch Space

> In some instances, your job may need to write files to disk temporarily while it's processing. You can use the /mbl/share/scratch folder for this. The advantage of using this folder is that it won't count towards your overall disk quota. Files in this folder are deleted automatically after 21 days. However, it's a good idea to delete them manually once you're done.

# Data Backups 

Data is stored on the Valentine Filesystem A.K.A GPFS or Scale

The following table describes the backup schedule for the different areas on Valentine.

| Area | Backup Schedule |
| --- | ---|
| Personal Workspace, Group Workspace, Instrument Folders | Every two days, Hourly snapshots for up to three days |
| Shared Software Folder | Every two days, no snapshots |
| Scratch space | Not backed up. Data is deleted automatically after 21 days |

# How to move files around the server using rsync

`rsync <originfolder>/<filename> <username>@hpc-head-002:/<destination_folder>`

`rsync /gpfs/nhmfsa/bulk/share/data/mbl/share/workspaces/groups/your_folder patrh5@hpc-head-002:/home/


# How to install software

You can install software in:

1. Your home folder
2. In a singularity container image
3. The shared software area (software must be compiled)

Each method has pros and cons

| Location | Pros | Cons |
| --- | --- | --- |
| Home directory | Easy with tools like Conda | Limited to 50GB, only usable by you |
| Singularity Container Image | Allows you to create an isolated environment with all the software dependencies you need, independent of other software on the cluster. Makes for a consistent and reproducible software environment. Sharable between different HPCs. | |
| Shared Software Folder | No limits on space, other uses can also use the software | You must compile the software, so it's more complicated. |

# How do I install Software in my Home Folder?

The easiest method of installing software is in your home folder using conda.

## Install Conda

You can download conda from https://docs.conda.io/en/latest/miniconda.html. Follow the instructions below to install it.

1. Navigate to your home directory
2. Download miniconda directly to the folder: `wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh`
3. Make the file executable: `chmod +x Miniconda3-latest-Linux-x86_64.sh`
4. Run this file to install miniconda: `chmod +x Miniconda3-latest-Linux-x86_64.sh`
5. Answer 'yes' to any questions, accept default options, and wait for installation to complete. Once done, restart your terminal. You should see base 

## Install software with conda

Let's use conda to install R

```
conda create -n r-environment r-essentials r-base
```

```
conda activate r-environment
```

```
which R
R
```

## Install software using a Singularity / Apptainer

Singularity, sometimes called Apptainer, is a useful tool that allows you to install software inside an isolated environment known as a **container**.

This is a similar concept to **docker**. Singularity is commonly used on HPC systems instead of docker, as singularity can be run without admin rights. 

## Create a singularity image

You can either install singularity on your own computer. Alternatively you can use our software-building server hpc-sw-003, which already has it installed. Please contact TS to request access to this server.  

`ssh <username>@hpc-sw-003`

Creat your singularity image file. It should end with file type .def:  

`touch my_image_file.def`

Open it with a text editor and construct your file:

```
Bootstrap: docker
From: ubuntu:22:04

%post
	apt update
	apt-get install bwa samtools -y

%help
	This is an Ubunter singularity with bwa and samtools installed.
```

The Bootstrap specifies using the ubuntu:22.04 image from the dockerhub container registry.  

The %post section allows you to list commands to run when building the image. In this example we are installing bwa and samtools applications inside the image.  

The %help label describes the container contents and what it does.  

Now that you have your definition file you can build the image. This will download the base image (ubuntu in this example), install the software (bwa and samtools) and create an image file ending in .sif:

`singularity build my_singularity.sif my_image_file.def

Now that we have the image  file, you can run your container using the following command. This is useful to see what you've created, as you can explore the file system and see what's installed.

` singularity run my_singularity.sif`

Now that you have your image, you can upload it to the HPC cluser using scp

`scp my_singularity.sif patrh5@hpc-head-002:~`

This will send it to your home directory 😊!

## In the shared software area

Until I need to use this, i'll save time typing it out. See the HPC user guide 7.3.1 for this info 😊.

You can check which software has already been installed in the shared software area:

`cd /mbl/share/software`
`ml avail`

The spider command gives you even more info
`ml spider`

To load this software onto your environment

`ml <software_name>/<version>

*When you go to the folders /software/testing/ or /software/common/, you may find more names, but if they do not appear when you run any of the commands above, they haven’t been properly installed.*

# How to use the job scheduler (Slurm)

Okay... this is why we're here. 

Now you have all the files and programs you need, but there’s a problem: lots of people are running huge data analyses simultaneously. That’s why job scheduling systems have been developed: to ensure a fair distribution of the computing resources across all users and to allow the cluster administrators to manage such resources. 

HPC uses a standard open-source job scheduling system called Slurm. A complete user guide can be found on their [website](https://slurm.schedmd.com/tutorials.html) and ideally you should become familiar with it. 

# Partitions / queues explained

The cluster has four 'partitions' or 'queues' where you submit your analyses as jobs. The decision here is the time and computational requirement of your jobs: hour, day, week, and month. As a general rule, the shorter the job, the less computationally expensive, the faster it gets asigned. Long analyses have low priority.

| Partition | Use  case |
| --- | --- |
| Hour | Very short jobs, or things that can be sped up to less than hour by multithreading |
| Day | Average jobs, or things that even with multithreading  can take up to 24 hours |
| Week | Long jobs that could take up to seven days, either because they cannot be parallelised or because they have low priority |
| Month | Non-important jobs that can run for a long time on low priority |

> Additionally, there is an interactive partition, that can be used to test computationally intensive tasks (almost any job really) to see how they perform, with a strict control of the used resources. There are many ways of using interactive, but it should not be abused as it can reduce the priority assigned to your future analyses. This is because the scheduling system calculates the priority of your jobs depending on how much you’re using the cluster, and usually when you use an interactive session you’re requesting a lot of resources (most of which you aren’t even using).

# How to submit a job
The first step is to create a file that contains all the instructions for the job schduler to run the analysis. 

This includes:
1. Specifying how to run the job
2. Install or activate any softare
3. Run a premade script

## #SBATCH lines allow you to define various options for your job

Most of these options are not required and there are many more than listed below, but they are important for optimization when the cluster gets busy and lots of people are submitting jobs to the same partition.

|#SBATCH command | Function |
| --- | --- |
| -J | Job Name |
| -p | Partition (hour, day, week, month) |
| -t | Time Limit, at what time does the partition end |
| --mem | minimum memory in MB |
| -c | number of CPUs |
| -e | file for standard error |
| -o | file for standard output |
| -w | name of nodes to run, e.g. hpc-gpu-001 |
| --mail-user | who to send email notification for job state changes |
| --mail-type | notify on state change: BEGIN, END, FAIL, ALL |

More options [here](https://slurm.schedmd.com/sbatch.html)

These lines are required if you are using conda from your home directory:
`source ~/miniconda3/etc/profile.d/conda.sh`
`conda init bash`

This line allows you to activate your conda environment called 'base':
conda activate base

The echo commands show how you can add additional output for your job

```
echo "Starting at 'date'"
echo "Running on hosts: $SLURM_NODELIST"
echo "Running on $SLURM_NNODES nodes"
echo "Current working directory is 'pwd'."
```

Finally, the actual command that you're running, in this example i'm using python to run a script called Python_Script.py, which is located in the same directory as my slurm_script.sh. You  **have** to write srun before your command so the scheduler knows you're queueing.

`srun python python_script.py`

## Let's put this all together, in a nano file called 'slurm_script.sh'

First open Nano, a word editor, to write your script on the command line:

`nano slurm_script.sh`

> The first line of your script is always #!/bin/bash, this is called a **she-bang** and it specifies the start of a bash script.  The shell’s name is an acronym for Bourne-again shell. Bash can execute the vast majority of scripts and thus is widely used because it has more features, is well developed and better syntax.

```
#!/bin/bash 
#SBATCH -J job_title
#SBATCH -p day
#SBATCH -t 00:30:00
#SBATCH --mem=2048
#SBATCH -c 4
#SBATCH -e /home/job_folder/job.%J.err
#SBATCH -o /home/job_folder/job.%J.out
#SBATCH -w hpc-gpu-001
#SBATCH --mail-user=p.hooper@nhm.ac.uk
#SBATCH --mail-type=ALL

source ~/miniconda3/etc/profile.d/conda.sh
conda init bash
conda activate base

echo "Starting at 'date'"
echo "Running on hosts: $SLURM_NODELIST"
echo "Running on $SLURM_NNODES nodes"
echo "Current working directory is 'pwd'"

srun python python_script.py
```

***now we have our script it's time to submit our job!***

Use sbatch to submit your job:

`sbatch slurm_script.sh`

 > If your analysis produces intermediate files between steps, you should set up a scratch folder for temporary files at /mbl/share/scratch. Ideally, clean up after your job. Scratch is set to auto-delete files after 21 days.

## Example 2

```
#!/bin/bash
#SBATCH -partition=hour
#SBATCH --output=myjob%J.out
#SBATCH -t 01:00:00
#SBATCH --mem-per-cpu=2G
#SBATCH --cpus-per-task=2

ml adapterremoval

NAME=$1

srun AdapterRemoval --file1 data/${NAME}_1.fastq.gz --file2 data/${NAME}_2.fastq.gz --threads $SLURM_CPUS_PER_TASK --gzip --basename ${NAME} --m 3 --minlength 25 --trimns --trimqualities --collapse --output1 analyses/adapterremoval/${NAME}_trimmed_R1.fastq.gz --output2 analyses/adapterremoval/${NAME}_trimmed_R2.fastq.gz --outputcollapsed analyses.adapterremoval/${NAME}_trimmed_R1R2.fastq.gz --outputcollapsedtruncated analyses/adapterremoval/${NAME}_truncated.fastq.gz --discarded analyses/adapterremoval/${NAME}_discarded.fastq.gz --singleton analyses/adapterremoval/${NAME}_singleton.fastq.gz --settings analyses/adapterremoval/${NAME}.settings
```

This time, isntead of using conda, i'm using the **ml** command to locate the [AdapterRemoval](https://adapterremoval.readthedocs.io/en/stable/) software from the shared software area:

`ml adapterremoval

Also notice, how instead of having a fixed sample ID, you can add variables to your .sh file. The variable NAME allows you to make the shell script more flexible, as you do not have to modify it any time you have a different sample name:

`NAME=$1

Once the slurm file is ready, you just need to submit the job:

`sbatch slurm_script.sh SRR001

## Using a 'for' loop to submit several jobs simultaneously:

```
for i in $(cat data/runids.txt); dp sbatch slurm_script.sh $i; done
```

## Example 3 - Using a singularity container

```
#!/bin/bash
#SBATCH --partition=hour
#SBATCH -c 2
#SBATCH -e /home/test/logs/job.%J.err
#SBATCH -o /home/test/logs/job.%J.out

srun singularity run \ 
--bind /workspaces/groups/rob-project-2:/workspaces/groups/rob-project-2 \
~/audiowaveform.sif \
audiowaveform --version
```

The srun line significes that we're going to run a singularity container in slurm. By default singularity can only access data in your home folder. However, you can use the --bind option to mount additional folders inside the container. This would be useful if, for example, the job needed to read or write data in the /workspaces/groups/rob-project-2 folder.

The line `~/audiowaveform.sif` refers to the singularity image file. 

The application here is audiowaveform

# Viewing and managing your jobs

You can use squeue to view ther status of jobs in the queue

`squeue -l`

**While the job is running** you can use scontrol to check its status:

`scontrol show jobid <job_id>`

**Once your job is finished**, you can use sacct to view details of your job. This is very useful information, because it will allow you to define the exact requirements of your jobs and make more efficient use of the computational resources!

`sacct --format=User,JobID,Jobname,partition,state,time,start,end,elapsed,MaxRss,MaxVMSize,nnodes,ncpus,nodelist -j <job_id>

You can use sinfo to check the status of the cluster nodes:

`sinfo -l`

Other states you may see:

- Alloc - this means the node is fully utilized by running jobs, so new jobs must wait in the queue
- Drain - this means the node is unable to receive new jobs, either due to an error or scheduled maintenance


**You can cancel** your jobs with:

`scancel <job_id>

**You can look at your own jobs** in the queue:

`squeue -u patrh5`

# Arraying Jobs in Slurm

The more advanced way to use slurm is to array your jobs. When doing loads of repetitive sequential analysis, this is a more effienct approach than having individual ones.

https://slurm.schedmd.com/job_array.html

# Snakemake

A python inspired workflow / pipeline management system to create reproducible and scalable data, creating contained environments that you can later deploy and execute in systems with different configurations 

https://snakemake.readthedocs.io/en/stable/