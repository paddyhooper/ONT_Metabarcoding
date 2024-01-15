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
| --- | --------|
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
{
conda create -n r-environment r-essentials r-base
}
```

```
{
conda activate r-environment
}
```

```
{
which R
R
}
```

## Install software using a Singularity / Apptainer

Singularity, sometimes called Apptainer, is a useful tool that allows you to install software inside an isolated environment known as a **container**.

This is a similar concept to **docker**. Singularity is commonly used on HPC systems instead of docker, as singularity can be run without admin rights. 

### Create a singularity image

You can either install singularity on your own computer. Alternatively you can use our software-building server hpc-sw-003, which already has it installed. Please contact TS to request access to this server.  

`ssh <username>@hpc-sw-003`

Creat your singularity image file. It should end with file type .def:  

`touch my_image_file.def`

Open it with a text editor and construct your file:

```
{ Bootstrap: docker
From: ubuntu:22:04

%post
	apt update
	apt-get install bwa samtools -y

%help
	This is an Ubunter singularity with bwa and samtools installed.
}
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

This will send it to your home directory :joy:!

## In the shared software area

Until I need to use this, i'll save time typing it out. See the HPC user guide 7.3.1 for this info :joy:.

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




#!/bin/bash
#SBATCH -J
#SBATCH -p
