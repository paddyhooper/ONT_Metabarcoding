# Organising ONT sequencing run

This document covers the preparation of my samples directly from the MinKNOW software.

During sequencing, the MinKNOW completed ultra high accuracy basecalling and barcoding of the samples, meaning the samples are separated into individual barcode files.

The sequencing process further separates the individual barcode files into smaller subfolders containing 1000 fastq read files. 

## Outputs from an ONT sequencing run

The files are organising into one folder named during the sequencing run. I transferred my folder into our group workspace on the NHM HPC. To navigate here, follow this path:

`workspaces/groups/jungblutlab/Paddy_Nanopore/

In my folder you can then find the two sequencing runs, ONT_1_18S (which contains the 18S data for the environmental microbiomes chapter (except soil, deep mats and WHIS)) and ONT_2 (which contains the nutrient experiment data, soil, deep mat biofilms, and WHIS1)

Never touch these folders! I am going to copy the .fastq files for the samples I want to use for my two projects into my home directory. 

Contents of each folder:

1. fastq_fail
2. fastq_pass
3. fast5_fail / pod5_fail
4. fast5_pass / pod5_pass
5. report.html
6. report.json
7. report.md
8. barcode alignment file
9. final summary
10. pore activity data
11. sequencing summary
12. throughput summary

The fast5 or pod5 files are your sequenced reads before they are barcoded.

The fastq files are the ones we use going forward (if we're happy with the barcoding job that MinKNOW did).


## Objectives

1. Make directory in your home directory containing the 
Barcode files need to be renamed by sample
2. Rename your files with sample ID using a reference file
3. Move all fastq files into one folder for each sample
4. Filter out short reads from our files
5. Trim the ONT adapters off
6. Mask low-complexity regions?
7. Perform taxonomic classification
8. Generate a Krona plot of taxonomy in your samples

## 1. Make a directory for my two sample sets
Login to your home directory on HPC-HEAD-002 through PuTTY and make two directories for your two projects called *ward_hunt_18* and *nut_exp*
*You will need your authenticator app for this*

`ssh patrh5@hpc-head-002
`mkdir ~/wh_18S ~/nut_exp ~/nut_exp/18S ~/nut_exp/16S
`ls

**Within each folder I also need to make a sub-folder for the two sequencing runs, as at the moment, the barcodes could still get mixed up until they're renamed.**

You should now have two directories in your home folder

We then need to transfer the required samples into each folder. We can do this using the command line.

First, I had to make a metadata file for each project in Excel and upload these to my server. These metadata files are *wh_18S_metadata* and *nut_exp_metadata*.

The metadata file contains the ID, sample ID, location, sample type, ont_run, ont_barcode, fmol, read_depth, temp, pH, conductivity, and sampling date

For wh_18S_metadata, a few samples were sequenced across both runs, for these samples I will make two entries for now and name the second sequencing runs with a 'b' prefix, e.g. WHL_M2 and WHL_M2b (for the second sequencing run).


## 2. Transferring my barcoded sequencing files to my home directory
The next step is to copy across the files I want in my home directory, in this case i am going to first copy all of the .fastq files across from both ONT_1_18S and ONT_2 in to two separate folders on my home directory.

`mkdir ~/ont1 ~/ont2

For reference, your raw ONT files and a zipped version of the folder are stored at:

/mbl/share/workspaces/groups/jungblutlab/Paddy_Nanopore/

As we are within the same servers I am just going to copy my files between the folders using

`cp -r` the -r flag tells the cp command to copy recursive, i.e. the focontents of the folder

Copy across both the fastq files from each sequencing run

`cp -r /mbl/share/workspaces/groups/jungblutlab/Paddy_Nanopore/ONT_1_18S/PH_2023-09-06_ARCTIC_18S/full_length_18S_rRNA_gene_reads_run_1/20230906_1808_X4_FAX39760_9alf312d/fastq_pass ~/ont1_fastq/

This long code is just copying from our complicated root path to the fastq pass files in our group workspace to the folder i just made in my home directory.

I repeated this step for the second ONT run. 

## 3. Organising my files by chapter
The next step is to copy across only the desired barcode files for each sequencing chapter into my two project folders.

I quickly made sub-folders in each project folder for the two sequencing runs 'ont1' and 'ont2' because some of the barcode numbers will match across the two runs. 

Due to the complexity of this job, i may do this manually in the command line using the 'cp' command one at a time, there's definitely a smart way but i want to be sure. 

For the wh_18S, all the barcodes can go into ont1, so i can speed this up by copying the whole file using:

`cp -r ~/ont1_fastq/fastq_pass/barcode* ~/wh_18S/ont1

For the rest of the barcode samples, I just copied them one by one using:

`cp -r ont2_fastq/fastq_pass/barcode40 ~/nut_exp/16S/ont2/

*I couldn't use the '[]' wildcards like I wanted to as you can't use this on parent directories (as the barcode folders contain the .fastq folders).*

I will cross reference the barcode numbers to the metadata excel spreadsheets I created.

## *Sidenote: Uploading and downloading files directly to your PC with SCP and proxyjump*

We can use WinSCP to easily move files between our server and our local PC. The benefit of WinSCP is the GUI means we can drag and drop between servers! In WinSCP we can also rename files manually, which can be a life saver.

However, WinSCP is (1) glitchy and likes to chuck you out unexpectedly, (2) slow and often times out when transferring large files.

It would be neater to use SCP directly in the command line. This is slightly complicated by the orca proxy to access the NHM servers. 

Normally, an SCP command looks like:

`scp /path/to/file.txt username@destination_server:~/my_directory

However, as we have the added Orca step we write the SCP command like:

`scp -J 'patrh5@orca.nhm.ac.uk' /path/to/file.txt patrh5@hpc-head-002:~/` - This would send the file to my home directory

To **download** files from the HPC to my desktop we flip the commands:

`scp -J 'patrh5@orca.nhm.ac.uk' patrh5@hpc-head-002:~/my_file.txt .`

Use the '-r' option to transfer a directory

As an example, here is the command I used to scp my zipped ONT run to my personal PC desktop in one command!:

`scp -r -J 'patrh5@orca.nhm.ac.uk' patrh5@hpc-head-002:/mbl/share/workspaces/groups/jungblutlab/Paddy_Nanopore/ONT_1_18S/PH_2023-09-06_ARCTIC_18S.tar.gz .\Desktop\


# 4. Combining sub-folders of .fastq files and renaming folders to my sample IDs

Now i have the correct samples for each project in specific project folders on my home directory, I need to create a bash script (.sh) that converts my numbered barcode folders into sample barcodes.

mkdir single_fastq_barcodes/
bash combine_fastqs.sh

## Each line of this script needs to look like

`cat ~/nut_exp/16S/ont2/barcode01/*.fastq.gz >> Single_fastq_barcodes/YOUR_SAMPLE_ID_FOR_BARCODE01.fastq.gz

## First, let's practice running a simple bash script on the job sceduler 
I think it's time i write a job script...

Let's test something super basic like..

`nano practice_slurm.sh`

in this text bash script i'm going to write

```
#!/bin/bash 
#SBATCH -J test_slurm
#SBATCH -p hour

echo "the job scheduler worked!"
```

Run the script using:
`bash practice_slurm.sh

This worked!

Now i am going to step it up a bit and try write a bash script that renames one file into another...

`echo "let's give it a go" >> before.txt

Now let's write a brief slurm script that changes before.txt to after.txt
`nano before_after.sh

copy this into the bash script in nano
```
#!/bin/bash 
#SBATCH -J file_name_change
#SBATCH -p hour

mv ~/before.txt ~/after.txt
echo "the file name has been changed"
```
run the script

`bash before_after.sh

this also worked!

## Second, let's try write the script for one practice change

don't panic, you have backups of your files so just give it a go!

Starting with one 16S file, let's see if we can do two things

1. Combine all the sub-folders into one single fastq barcode
2. Rename the .fastq file now containing all your .fastq files by sample ID

Let's try it! First let's make a directory called 'single_fastq_barcodes/' in the 'nut_exp' project folder.

`mkdir ~/nut_exp/single_fastq_barcodes

Next we write a  script to convert only one barcode for now into a single file:

The folder / sample I chose is 'barcode01' from the second sequencing run, ont2, AKA sample 'M_N4_16S'. 

***Note: This sample contained 140319 reads and 117 subfolders!***

`nano test_combine_fastqs.sh

In this script i copied the below text. This uses the cat (concatenate) function and a wildcard, the asterisk, to move all .fastq.gz files contained in the barcode01 folder to the 'single_fastq_barcodes/' directory we just made and combinese them, hopefully, into one '.fastq.gz' file. The '.gz' tells us that the file is zipped.

***Note: Notice how i specified the sample ID manually as 'M_N4_16S'

```
#!/bin/bash 
#SBATCH -J test_combine_fastqs
#SBATCH -p hour

cat ~/nut_exp/16S/ont2/barcode01/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_N4_16S.fastq.gz

echo "single file barcode created!"
```

Once you have saved this text inside your test_combine_fastqs.sh script, submit the job to the scheduler using:

`bash test_combine_fastqs.sh

It worked I think!!

## Third, let's make a script for each set of samples so we don't get confused.

I need scripts for:

wh_18S/ont1 = 96 samples
wh_18S/ont2 = 24 samples

nut_exp/16S/ont2/ = 40 samples
nut_exp/18S/ont2/ = 33 samples
nut_exp/18S/ont1/ = 8 samples

***The nice thing about scripts is I can basically just copy the one above :)***