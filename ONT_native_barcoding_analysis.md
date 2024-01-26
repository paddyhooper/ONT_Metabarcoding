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

*I couldn't use the '[]' [wildcard](https://tldp.org/LDP/GNU-Linux-Tools-Summary/html/x11655.htm) like I wanted to as you can't use this on parent directories (as the barcode folders contain the .fastq folders).*

I will cross reference the barcode numbers to the metadata excel spreadsheets I created.

___

## Note: Uploading and downloading files directly to your PC with SCP and proxyjump*

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

___

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

nut_exp/18S/ont1/ = 8 samples
nut_exp/16S/ont2/ = 40 samples
nut_exp/18S/ont2/ = 33 samples


wh_18S/ont1 = 96 samples
wh_18S/ont2 = 24 samples



***The nice thing about scripts is I can basically just copy the one above :)***

#### Script 1 - nut_exp_ont1

`nano combine_fastqs_nut_exp_ont1.sh

```
#!/bin/bash 
#SBATCH -J combine_fastqs_nut_exp_ont1
#SBATCH -p hour

cat ~/nut_exp/18S/ont1/barcode10/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_TZ3_18S.fastq.gz

cat ~/nut_exp/18S/ont1/barcode11/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_TZ2_18S.fastq.gz

cat ~/nut_exp/18S/ont1/barcode12/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_TZ1_18S.fastq.gz

cat ~/nut_exp/18S/ont1/barcode49/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_TZ1_18S.fastq.gz

cat ~/nut_exp/18S/ont1/barcode61/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_TZ4_18S.fastq.gz

cat ~/nut_exp/18S/ont1/barcode62/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_TZ3_18S.fastq.gz

cat ~/nut_exp/18S/ont1/barcode64/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_TZ2_18S.fastq.gz

cat ~/nut_exp/18S/ont1/barcode89/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/NC_18S.fastq.gz

echo "single file barcodes created!"
```

Run the script:

`bash combine_fastqs_nut_exp_ont.sh

#### Script 2 - nut_exp_ont2_16S

`nano combine_fastqs_nut_exp_ont2_16S.sh

```
#!/bin/bash 
#SBATCH -J combine_fastqs_nut_exp_ont2_16S
#SBATCH -p hour

cat ~/nut_exp/16S/ont2/barcode01/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_N4_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode02/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_N3_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode03/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_N2_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode04/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_N1_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode05/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_C4_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode06/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_C3_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode07/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_C2_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode08/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_C1_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode09/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_NP4_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode10/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_NP3_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode11/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_NP2_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode12/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_NP1_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode13/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_P4_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode14/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_P3_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode15/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_P2_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode16/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_P1_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode17/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_N4_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode18/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_N3_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode19/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_N2_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode20/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_N1_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode21/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_C4_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode22/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_C3_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode23/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_C2_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode24/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_C1_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode25/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_NP4_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode26/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_NP3_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode27/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_NP2_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode28/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_NP1_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode29/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_P4_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode30/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_P3_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode31/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_P2_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode32/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_P1_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode33/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/NC_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode34/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_TZ3_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode35/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_TZ2_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode36/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_TZ1_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode37/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_TZ4_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode38/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_TZ3_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode39/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_TZ2_16S.fastq.gz
cat ~/nut_exp/16S/ont2/barcode40/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_TZ1_16S.fastq.gz

echo "single file barcodes created!"
```
Run the script:

`bash combine_fastqs_nut_exp_ont2_16S.sh

#### Script 3 - nut_exp_ont2_18S

`nano combine_fastqs_nut_exp_ont2_18S.sh

```
#!/bin/bash 
#SBATCH -J combine_fastqs_nut_exp_ont2_18S
#SBATCH -p hour

cat ~/nut_exp/18S/ont2/barcode41/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_N4_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode42/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_N3_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode43/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_N2_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode44/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_N1_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode45/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_C4_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode46/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_C3_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode47/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_C2_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode48/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_C1_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode49/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_NP4_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode50/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_NP3_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode51/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_NP2_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode52/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_NP1_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode53/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_P4_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode54/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_P3_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode55/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_P2_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode56/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_P1_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode57/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_N4_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode58/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_N3_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode59/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_N2_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode60/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_N1_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode61/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_C4_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode62/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_C3_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode63/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_C2_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode64/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_C1_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode65/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_NP4_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode66/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_NP3_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode67/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_NP2_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode68/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_NP1_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode69/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_P4_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode70/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_P3_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode71/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_P2_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode72/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/W_P1_18S.fastq.gz
cat ~/nut_exp/18S/ont2/barcode93/*.fastq.gz >> ~/nut_exp/single_fastq_barcodes/M_TZ1_18Sb.fastq.gz

echo "single file barcodes created!"
```

Run the script:

`bash combine_fastqs_nut_exp_ont2_18S.sh

#### Script 4 - wh_18S_ont1

`mkdir ~/wh_18S/single_fastq_barcodes
`nano combine_fastqs_wh_18S_ont1.sh

```
#!/bin/bash 
#SBATCH -J combine_fastqs_wh_18S_ont1
#SBATCH -p hour

cat ~/wh_18S/ont1/barcode01/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_W2.fastq.gz
cat ~/wh_18S/ont1/barcode02/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_W1.fastq.gz
cat ~/wh_18S/ont1/barcode03/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_W3.fastq.gz
cat ~/wh_18S/ont1/barcode04/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_W2.fastq.gz
cat ~/wh_18S/ont1/barcode05/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_W1.fastq.gz
cat ~/wh_18S/ont1/barcode06/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_W3.fastq.gz
cat ~/wh_18S/ont1/barcode07/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_W2.fastq.gz
cat ~/wh_18S/ont1/barcode08/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_W1.fastq.gz
cat ~/wh_18S/ont1/barcode09/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W4.fastq.gz
cat ~/wh_18S/ont1/barcode10/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W3.fastq.gz
cat ~/wh_18S/ont1/barcode11/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W2.fastq.gz
cat ~/wh_18S/ont1/barcode12/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W1.fastq.gz
cat ~/wh_18S/ont1/barcode13/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/AP_W3.fastq.gz
cat ~/wh_18S/ont1/barcode14/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_W3.fastq.gz
cat ~/wh_18S/ont1/barcode15/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/AP_W2.fastq.gz
cat ~/wh_18S/ont1/barcode16/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/AP_W1.fastq.gz
cat ~/wh_18S/ont1/barcode17/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W12.fastq.gz
cat ~/wh_18S/ont1/barcode18/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W11.fastq.gz
cat ~/wh_18S/ont1/barcode19/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W10.fastq.gz
cat ~/wh_18S/ont1/barcode20/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W9.fastq.gz
cat ~/wh_18S/ont1/barcode21/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W8.fastq.gz
cat ~/wh_18S/ont1/barcode22/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W7.fastq.gz
cat ~/wh_18S/ont1/barcode23/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W6.fastq.gz
cat ~/wh_18S/ont1/barcode24/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_W5.fastq.gz
cat ~/wh_18S/ont1/barcode25/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_M1.fastq.gz
cat ~/wh_18S/ont1/barcode26/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/AP_M7.fastq.gz
cat ~/wh_18S/ont1/barcode27/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/AP_M6.fastq.gz
cat ~/wh_18S/ont1/barcode28/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/AP_M5.fastq.gz
cat ~/wh_18S/ont1/barcode29/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/AP_M4.fastq.gz
cat ~/wh_18S/ont1/barcode30/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/AP_M3.fastq.gz
cat ~/wh_18S/ont1/barcode31/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/AP_M2.fastq.gz
cat ~/wh_18S/ont1/barcode32/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/AP_M1.fastq.gz
cat ~/wh_18S/ont1/barcode33/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_M4.fastq.gz
cat ~/wh_18S/ont1/barcode34/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_M3.fastq.gz
cat ~/wh_18S/ont1/barcode35/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_M2.fastq.gz
cat ~/wh_18S/ont1/barcode36/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_M1.fastq.gz
cat ~/wh_18S/ont1/barcode37/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_M5.fastq.gz
cat ~/wh_18S/ont1/barcode38/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_M4.fastq.gz
cat ~/wh_18S/ont1/barcode39/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_M3.fastq.gz
cat ~/wh_18S/ont1/barcode40/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_M2.fastq.gz
cat ~/wh_18S/ont1/barcode41/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WHIS2_M1.fastq.gz
cat ~/wh_18S/ont1/barcode42/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WHIS1_M3.fastq.gz
cat ~/wh_18S/ont1/barcode43/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_M5.fastq.gz
cat ~/wh_18S/ont1/barcode44/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_M4.fastq.gz
cat ~/wh_18S/ont1/barcode45/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_M3.fastq.gz
cat ~/wh_18S/ont1/barcode46/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_M2.fastq.gz
cat ~/wh_18S/ont1/barcode47/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_M1.fastq.gz
cat ~/wh_18S/ont1/barcode48/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_M5.fastq.gz
cat ~/wh_18S/ont1/barcode49/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_M3.fastq.gz
cat ~/wh_18S/ont1/barcode50/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_M2.fastq.gz
cat ~/wh_18S/ont1/barcode51/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_M1.fastq.gz
cat ~/wh_18S/ont1/barcode52/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WHIS3_M3.fastq.gz
cat ~/wh_18S/ont1/barcode53/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WHIS3_M2.fastq.gz
cat ~/wh_18S/ont1/barcode54/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WHIS3_M1.fastq.gz
cat ~/wh_18S/ont1/barcode55/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WHIS2_M3.fastq.gz
cat ~/wh_18S/ont1/barcode56/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WHIS2_M2.fastq.gz
cat ~/wh_18S/ont1/barcode57/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_M4.fastq.gz
cat ~/wh_18S/ont1/barcode58/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_M3.fastq.gz
cat ~/wh_18S/ont1/barcode59/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_M2.fastq.gz
cat ~/wh_18S/ont1/barcode60/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_M1.fastq.gz
cat ~/wh_18S/ont1/barcode61/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_M7.fastq.gz
cat ~/wh_18S/ont1/barcode62/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_M6.fastq.gz
cat ~/wh_18S/ont1/barcode63/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_M5.fastq.gz
cat ~/wh_18S/ont1/barcode64/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_M4.fastq.gz
cat ~/wh_18S/ont1/barcode65/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_DM3.fastq.gz
cat ~/wh_18S/ont1/barcode66/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_DM2.fastq.gz
cat ~/wh_18S/ont1/barcode67/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_M10.fastq.gz
cat ~/wh_18S/ont1/barcode68/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_M9.fastq.gz
cat ~/wh_18S/ont1/barcode69/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_M8.fastq.gz
cat ~/wh_18S/ont1/barcode70/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_M7.fastq.gz
cat ~/wh_18S/ont1/barcode71/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_M6.fastq.gz
cat ~/wh_18S/ont1/barcode72/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/QL_M5.fastq.gz
cat ~/wh_18S/ont1/barcode73/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_C5.fastq.gz
cat ~/wh_18S/ont1/barcode74/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_C4.fastq.gz
cat ~/wh_18S/ont1/barcode75/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_C3.fastq.gz
cat ~/wh_18S/ont1/barcode76/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_C2.fastq.gz
cat ~/wh_18S/ont1/barcode77/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_C1.fastq.gz
cat ~/wh_18S/ont1/barcode78/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/LA_M3.fastq.gz
cat ~/wh_18S/ont1/barcode79/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/LA_M2.fastq.gz
cat ~/wh_18S/ont1/barcode80/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/LA_M1.fastq.gz
cat ~/wh_18S/ont1/barcode81/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_C3.fastq.gz
cat ~/wh_18S/ont1/barcode82/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_C2.fastq.gz
cat ~/wh_18S/ont1/barcode83/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_C1.fastq.gz
cat ~/wh_18S/ont1/barcode84/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_C5.fastq.gz
cat ~/wh_18S/ont1/barcode85/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_C4.fastq.gz
cat ~/wh_18S/ont1/barcode86/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_C3.fastq.gz
cat ~/wh_18S/ont1/barcode87/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_C2.fastq.gz
cat ~/wh_18S/ont1/barcode88/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_C1.fastq.gz
cat ~/wh_18S/ont1/barcode89/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/NC.fastq.gz
cat ~/wh_18S/ont1/barcode90/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_S5.fastq.gz
cat ~/wh_18S/ont1/barcode91/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_S3.fastq.gz
cat ~/wh_18S/ont1/barcode92/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_S2.fastq.gz
cat ~/wh_18S/ont1/barcode93/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_S1.fastq.gz
cat ~/wh_18S/ont1/barcode94/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_S1.fastq.gz
cat ~/wh_18S/ont1/barcode95/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_C5.fastq.gz
cat ~/wh_18S/ont1/barcode96/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_C4.fastq.gz

echo "single file barcodes created!"
```

Run the script:

`bash combine_fastqs_wh_18S_ont1.sh

#### Script 5 - wh_18S_ont2

`nano combine_fastqs_wh_18S_ont2.sh

```
#!/bin/bash 
#SBATCH -J combine_fastqs_wh_18S_ont2
#SBATCH -p hour

cat ~/wh_18S/ont2/barcode73/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_S3.fastq.gz
cat ~/wh_18S/ont2/barcode74/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_S2.fastq.gz
cat ~/wh_18S/ont2/barcode75/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_S1b.fastq.gz
cat ~/wh_18S/ont2/barcode76/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_S5.fastq.gz
cat ~/wh_18S/ont2/barcode77/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_S4.fastq.gz
cat ~/wh_18S/ont2/barcode78/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_S3.fastq.gz
cat ~/wh_18S/ont2/barcode79/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_S2.fastq.gz
cat ~/wh_18S/ont2/barcode80/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT1_S1.fastq.gz
cat ~/wh_18S/ont2/barcode81/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WHIS1_M2.fastq.gz
cat ~/wh_18S/ont2/barcode82/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WHIS1_M1.fastq.gz
cat ~/wh_18S/ont2/barcode83/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_S4.fastq.gz
cat ~/wh_18S/ont2/barcode84/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_S3b.fastq.gz
cat ~/wh_18S/ont2/barcode85/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_S2b.fastq.gz
cat ~/wh_18S/ont2/barcode86/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT3_S1b.fastq.gz
cat ~/wh_18S/ont2/barcode87/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_S5.fastq.gz
cat ~/wh_18S/ont2/barcode88/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WT2_S4.fastq.gz
cat ~/wh_18S/ont2/barcode89/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_DM3b.fastq.gz
cat ~/wh_18S/ont2/barcode90/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_DM2b.fastq.gz
cat ~/wh_18S/ont2/barcode91/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_DM1.fastq.gz
cat ~/wh_18S/ont2/barcode92/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_M5b.fastq.gz
cat ~/wh_18S/ont2/barcode93/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_M3b.fastq.gz
cat ~/wh_18S/ont2/barcode94/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_M2b.fastq.gz
cat ~/wh_18S/ont2/barcode95/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WH_M1b.fastq.gz
cat ~/wh_18S/ont2/barcode96/*.fastq.gz >> ~/wh_18S/single_fastq_barcodes/WHIS_M3b.fastq.gz

echo "single file barcodes created!"
```

Run the script:

`bash combine_fastqs_wh_18S_ont2.sh

# 5. Downstream analysis
So far, we have only used base BASH commands to change our files, this is a good thing! The simplest approach is always the best when it  comes to coding. 

For the specific bioinformatics analysis we will mainly use conda software. This requires the installation of conda on my local home directory, within conda we can then install a huge diversity of software packages and create environments for software to run in...

___

### Note: Conda

1. Conda is a package and environment manager that is the easiest way to install most bioinformatics tools. 
2. Conda handles different environments very nicely making life easier. e.g. sometimes software A has dependencies which include software B and software C, but then software C will depend on a different type of software B. Conda allows us to create and manage separate environments to avoid these types of version conflicts.
3. Conda helps with reproducibility. You don't need a specific version of packages to make some script work.

#### 1. Installing Conda

[Full breakdown here](https://astrobiomike.github.io/unix/conda-intro#binder-available) 

Conda comes in two broad forms: *Anaconda* and *Miniconda*. Anaconda is large with lots of programs already installed. Miniconda is more lightweight, meaning we can install just what we want. This is preferable to not clog up our home directory!

The download page for Miniconda is [here](https://docs.conda.io/projects/miniconda/en/latest/)

For the server, we want the Linux version as the server runs in Linux, even if you're logging in from a Windows PC etc.

Downloading Conda:

1. Navigate to your home directory
2. Download miniconda directly to the folder:
`wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh`
3. Make the file executable:
`chmod +x Miniconda3-latest-Linux-x86_64.sh`
4. Run this file to install miniconda:
`bash ./Miniconda3-latest-Linux-x86_64.sh
5. Answer 'yes' to any questions, accept default options, and wait for installation to complete. Once done, *restart your terminal*. You should now see base next to your command line: "(base) patrh5@hpc-head-002:~$"

**NOTE: Python is also installed as part of base Conda, but R is not!**

#### 2. Creating and navigating environments

Environments represent the infrastructure our computer is currently operating in. This includes like if certain variables exist and how they are set. 

**Base environmnet**  
The "base" conda environment is the home base inside conda. We wouldn't want to install lots of complicated programs here, as the more things added, the more likely something is going to end up having a conflict. But the base environment is somehwere we might want to install simpler programmes we tend to use a lot. 

**Making a new environment**  
The simplest way to create a new conda environment is like so:
`conda create -n new-env` '-n' specifying the environment name

**Activating / entering our new environment**
`conda activate new-env` You should now see the prompt has changed to "(new-env) patrh5@hpc-head-002:~$"

**List all your environments**
`conda env list

At the moment you should just see base

**Deactivating / leaving our new environment**
`conda deactivate` You should now see "base" next to your login once again

**Let's make an environment with a specific python version**
By default conda create command will use the python version that the base conda environment is running. 

Some software won't be built for the latest conda or has the requirement of a specific conda, here we can use environments to create a new environment with the correct python installed in one command:

`conda create -n python-v2.7 python=2.7
`#The environment is named python-v2.7 and contains python V2.7

We can compare our base and home by activating both.

In your home directory (where base automatically now runs), type
`python --version`
`#*Output: Python 3.11.5

In our new python environment
`conda activate python-v2.7
`python --version
`#Output: Python 2.7

**Remove an environment

`conda env remove -n python-v2.7


## Install software with conda
To install the packages we want, we use `conda install` command. But we need to tell conda where to look for it as conda stores packages in different "channels". 

1. Make an environment to house the software you want to install

`conda create -n prodigal_env

2. Let's make sure prodigal isn't already installed

`prodigal -v

3. A lot of conda for bioinformatics can be found in the channel "bioconda":

`conda install -c bioconda prodigal` note how the "-c" specifies look in this channel name

4. But wait! bioconda isn't that simple sadly. For many packages such as `prodigal`, this will work. For many packages from bioconda to install properly, more channels are needed to be set in a specific hierarchy. This is especially important for packages that may seem to install, but have internal errors that will mess up scripts down the line!

This is because even though prodigal may be in the bioconda channel, some of its dependencies may be found elsewhere or in multiple places, and having those 3 channels set in the proper priority helps to ensure the correct versions of everything are being used.

**The safest way to install is to specify your channels:**

`conda install -c conda-forge -c bioconda -c defaults package_name

5. Always worth checking you've definitely installed it in your environment rather than your home directory

`conda deactivate
`prodigal -v` if this returns no results, you haven't installed prodigal in your home directory or base conda :) 

6. To uninstall a package within an environment, we first

`conda activate <your_environment>
`conda uninstall <the_package_you_don't_want>
`<the_package_you_don't_want> -v

**Install a specific version of a package**
 
If we want to specify a different version of prodigal we need to know it exists in conda. We can see all the version types by searching

conda search -c bioconda prodigal

This tells us verion 2.6.2 and 2.6.3 are both available in bioconda, with different builds for each. A build specifies that the software hasn't changed but **how it was packaged for conda has!**

`conda install -c conda-forge -c bioconda -c defaults prodigal=2.6.2

**NOTE: Setting a pre-set specification of channels for our version of bioconda is preferential to specifying the order of channels every time.**

- When we do something in conda, it automatically searches our stored channels (and it does so in a specific order).
- Above, we had to specify -c biocoonda when we installed prodigal because that channel wasn't yet in our stored channels.
- We can either set these ahead of time, or we can specify them each time we install something.
- Most of the things we biologists will want to use are found in the [bioconda channel](https://bioconda.github.io/index.html)
- As a rule of thumb, it is best to follow the **bioconda documentation** and specify all the channels required for installing from bioconda in the proper order.

Option 1: We can specify these channels in each installation command like so:

`conda install -c conda-forge -c bioconda -c defaults prodigal

Option 2: we can set the channels ahead of time like the bioconda documentation demonstrates (when done this way, the last one we add has the highest priority):

```
conda config --add channels defaults # no need to worry about the warning
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict
```

Our channel ist are then shown in priority order:
`conda config --get channels

Therefore, we could now just run a much simpler command as below, as all the channels were pre-specified
`conda install prodigal

**V IMPORTANT - DO THIS IN YOUR DESIRED CONDA ENVIRONMENT, IT'S NOT ADVISABLE TO MAKE THESE CHANNELS IN YOUR HOME DIRECTORY**

## BONUS: mamba

Mamba is a drop-in replacement for conda that works to improve upon some aspects of the conda infrastructure. It often performs installations much faster and solve environments quicker.

We need to install mamba within conda. This is highly advisable for improving your life!

Note: Here I have specified the environment as "base" our base conda environment so that mamba is installed here.

`conda install -n base -c conda-forge mamba

Forevermore, we can use mamba in place of conda for *installing* things and *creating environments*. However, it is probably still for the best if we use conda when *activating / deactivating* environments.

`mamba create -n bit -y -c conda-forge -c bioconda -c defaults -c astrobiomike bit

Here:
"-n" specifies creating a new environment called "bit"
"-y" says to not ask for any confirmation
"-c..." each one specifies the appropriate channels in the appropriate order
"bit" the specific program name we want

Note: we still use conda to activate or deactivate

`conda activate bit

## BONUS 2: .yaml files

Sometimes it's convenient to insteas use a yaml file that holds all the information about the programs we want, and we can just give that file to the mamba/conda create command.

We can write these files, like our bash scripts, in nano:

`nano prodigal.yaml

And copy in this script:

```
name: py3.5-with-prodigal
channels:
    - conda-forge
    - bioconda
    - defaults
dependencies:
    - python=3.5
    - prodigal
```

"name:" specifies the name of the conda environment
"channels:" specify the order of the channels needed for installation, for the case of bioconda, they must be ordered in this order to specify the correct priority
"dependencies:" specifies the programs or versions we want within our environment, for this example we are specifying python version 3.5 and no version specified for prodigal.

We can then create our conda environment from the .yaml file using

`mamba env create -f prodigal.yaml #note how we included 'env' here`

"-f" denotes file


## Managing R and RStudio with conda

**Setting up a conda environment with R and RStudio**

More info [here](https://astrobiomike.github.io/R/managing-r-and-rstudio-with-conda)

`mamba create -n r-env -y -c conda-forge r-base r-essentials rstudio r-tidyverse #note I am installing r-essentials instead of r-base as it wasn't happy with me, as here: (https://www.biostars.org/p/450316/)! 

"-n" here is where we provide the name we want the environment to have, in this case naming it R with the version following
"-y" says not to ask us for any confirmation
"-c conda-forge" here is where we are specifying the one channel we need for the particular packages we are trying to install

**Opening RStudio in our conda environment**

Once we've created the above, we just need to activate the conda environment we created

`conda activate r-env
`conda list #this tells us what packages are in the environment`
`which R
`R #this allows us to run R
`q() #to quit R`

Then once in the environment, we can execute `rstudio` to open an RStudio window within that conda environment!

rstudio

**Installing specific R libraries**

R Libraries can be installed in the R conda environment the way we normally would (meaning no conda involved), but this can occasionally cause problems. Because of this, I generally try to find whatever R libraries I want as conda-installable packages and install them in my conda R environmnent. 

Note! You need to be in the activated R environment!

Once you are, find the R package page on the anaconda website and follow their instructions, e.g.

`mamba install -y -c conda-forge -c bioconda -c defaults bioconductor-dada2 #note how i once again specified all the bioconda channels in priority order (from right to left in terms of importance)

Then in the R environment within this conda environment, we are able to load the library with:

'R 
'library(dada2) 

**Alternatively [installing R libraries](https://docs.anaconda.com/free/anaconda/packages/using-r-language/)

When using conda to install R packages, you will need to add r- before the regular package name. For instance, if you want to install dada2, you will need to use `conda install r-dada2`

Remember to do this in an environment within your r-env, i.e. maybe make a new environment for certain jobs or tasks.

next step - decide what to do with my data!


## Quality control

For larger processing steps it can be advantageous to make a temp directory in the scratch space. Here we make a temp directory for our commands we submit in our job scripts "paddy"

`ls /mbl/share/scratch
`mkdir /mbl/share/scratch/paddy

https://help.cropdiversity.ac.uk/slurm-overview.html 

Let's come back to that...

Now let's work out what to do!!

Maria is re-high accuracy version using dorado instead of guppy
Use a GPU on the HPC rather than a CPU


Nanostat - quality check data
PoreChop for removing barcodes
Nanoplot 

Nanopore already does pass and fail, in illumina we have to do that in s 
QC 
Remove barcodes
apply length filtering to my reads  - remove chimeras as much as possible - seqtk - https://github.com/lh3/seqtk or https://bioinf.shenwei.me/seqkit/ 
loop through your files and run it on everything 

map anything to human reads and remove them
remove frog host reference genome



#EPI2ME Labs wf-16S

[GitHub page](https://github.com/epi2me-labs/wf-16s/tree/master?tab=readme-ov-file#FAQs)

The 16S and 18S workflow for taxonomic classification of single reads from amplicon-targeted sequencing. 

This workflow allows you to taxonomically classify 16S rRNA and 18S rRNA amplicons using default or custom databases.

The default databases include:

NCBI targeted loci: 16S rDNA, 18S rDNA, ITS #however this 18S data looks specified for fungi, [here](https://www.ncbi.nlm.nih.gov/refseq/targetedloci/)

Two approaches are available: `minimap2` using alignment, default option or `kraken`, k-mer based.

Results include:
- An abundance table with counts per taxa in all samples
- Interactive sankey and sunburst plots to explore different identified lineages
- A barplot comparing the abundances of most abundant taxa in all samples

## Install

This workflow uses nextflow to manage compute and software resources. Therefore, nextflow will need to be installed before attempting to run the workflow.

Therefore, let's first install [nextflow](https://www.nextflow.io/)

```
cd ~
java -version

#Output:
#openjdk version "11.0.13" 2021-10-19
#OpenJDK Runtime Environment JBR-11.0.13.7-1751.21-jcef (build 11.0.13+7-b1751.21)
#OpenJDK 64-Bit Server VM JBR-11.0.13.7-1751.21-jcef (build 11.0.13+7-b1751.21, mixed mode)

#create a nextflow directory in the home directory
curl -s https://get.nextflow.io | bash
ls #you should now see nextflow

#test your nextflow is installed
 ./nextflow run hello
```

***Note: 19.01.23: We also need to have docker or singularity installed. I tried installing docker but I wasn't able to as I don't have sudo access. Therefore, I may have to try a singularity. Going to instead try carry on and see if it works, i.e. singularity is already installed for me?***

Now we have nextflow installed, we can use it to install the wf-16S nextflow pipeline directly from epi2me-labs

`./nextflow run epi2me-labs/wf-16s --help 

A demo dataset is provided for testing of the workflow. It can be downloaded using: 

`wget https://ont-exd-int-s3-euwst1-epi2me-labs.s3.amazonaws.com/wf-16s/wf-16s-demo.tar.gz
`tar -xzvf wf-16s-demo.tar.gz

We have downloaded a zipped file wf-16S-demo.tar.gz and then unzipped it creating a new directory 'wf-16S-demo', in this directory we have a file test_data containing three barcodes and an unclassified folder

We can then run the workflow using the demo data using:

```
nextflow run epi2me-labs/wf-16s \
--fastq wf-16s-demo/test_data/ \
-profile standard 

#the backslash tells us to read the next line as the same command
# the -fastq is telling us where to find the data, in the directory we just downloaded
# the [-profile standard](https://labs.epi2me.io/wfquickstart/) is saying to use docker
```

when i searched `which singularity` it came up in the local bin, so it might be pre-installed

Let's just try run the command, it doesn't work it's because of no docker installed:

output:

`Command error:
`  .command.run: line 290: docker: command not found

Therefore, we need to work out how to run it with singularity. It doesn't tell us on the tutorial how to specify a singularity which is a pain.

Let's try

```
~/nextflow run epi2me-labs/wf-16s \
--fastq wf-16s-demo/test_data/ \
-profile singularity 
```

This also doesn't work as I don't have singularity

I can't seem to use sudo to install, therefore I might try install singularity through conda? In my base conda, I will create an environment 'singularity-env':

`mamba create -n singularity-env -y -c conda-forge singularity

now let's try activate our new environment:

`conda activate singularity-env #I now have (singularity-env) patrh5@hpc-head-002:~$

Now let's try re-run the nextflow command

```
~/nextflow run epi2me-labs/wf-16s \
--fastq wf-16s-demo/test_data/ \
-profile singularity 
```

Now we need to update java... (remember this will only do it in this 'singularity-env' environment)

`conda install bioconda::java-jdk

I can't do this either, as java isn't working now...


## [NHM London Nextflow Guide](https://github.com/NHM-London-Bioinformatics/pipelines-sop0

### Top tips for running a pipeline

- Always run a pipeline for a specific set of data in its own working directory.
- Nextflow does most of its work in a specific `work` directory that you specify, it also creates a `.nextflow` and `.nextflow` log files in the current working directory, and these may clash with other nextflow directories running in the same directory. 
- Always run a pipeline using screen. Nextflow dispatches slurm jobs while the main program continues running and reporting progress, but it can take hours so being able to close the screen is invaluable
- Always specify the version of the pipeline `-r` when running a nextflow pipeline. When you want to compare a new run to a previous dataset, use the same version. Otherwise, we generally recommend running the most recent and often most stable version.
- Always keep your own record of the exact command you run, so you can replicate the analysis with future data.
- All pipelines can take as input a sample sheet in TSV form. These can be created in Excel or Google Sheets. This is their recommendation for inputting sample information.
- Always use absolute file paths. 

### Queue selection and other configurations

We have written a configuration file that optimises how nextflow runs on the NHM HPC, including dispartching jobs with Slurm. 

The main thing to remember is that this allows us to specify which Slurm queue to run with. 

When running any of these piplines on the NHM HPC, you must include the NHM config file as follows:

`-c /mbl/share/workspaces/groups/nextflow/config/NHM.config`

This is included in all of the example commands for the pipelines. 

The config file creates a set of default parameters that instruct the pipeline to dispatch all jobs to a specific slurm queue. Briefly, this config uses the `genericcpuhour` profile, dispatching jobs to the CPU nodes on the hour queue. If your dataset is large and the pipeline needs mroe time, ,you can change to the day, week or month by just swapping out `genericcpuhour` for `genericcpuday`, etc.

In the ampliseq workflow, they run their nextflow command with the `-profile` parameter: 

`-profile genericcpuhour, singularity \

This looks like what I need to specify

Here's an example from the ampliseq site:

```
nextflow run nf-core/ampliseq -r <version> \
  -c /mbl/share/workspaces/groups/nextflow/config/NHM.config \
  -profile genericcpuhour,singularity \
  -w work \
  --outdir out \
  --input </path/to/samplesheet.tsv> \
  --FW_primer <FORSEQ> \
  --RV_primer <REVSEQ> \
  --cutadapt_min_overlap 3 \
  --trunclenf <FTRUNC> \
  --trunclenr <RTRUNC> \
  --max_ee 1 \
  --min_frequency 2 \
  --dada_ref_taxonomy "pr2=4.14.0" \
  --cut_dada_ref_taxonomy \
  --skip_diversity_indices --skip_ancom --skip_barplot --skip_alpha_rarefaction \
  --email <you@nhm.ac.uk>
 ```


## NHM HPC nextflow configuration files

The NHM have written configuration files that optimize the nextflow pathway on the NHM HPC. 

Info on their configuation is available here:

### NHM.config

This config sets the path to the NHM [singularity cache](https://github.com/NHM-London-Bioinformatics/pipelines-sop/blob/main/singularity_cache.md) directory and specifies that all processes should be executed by slurm. It then sets four profiles, corresponding to the four different NHM queues, all using CPUs rather than GPUs. As the NHM HPC slurm configuration is set to dispatch to CPU by default, as then no specific CPU dispatch instructions are included. 

**Queue specification**
The config file specifies a map (aka dict or hash) that comprises 3 unique queue options, such that successive attempts will be dispatched to queues with longer running times. Namely, attempts 1 and 2 are dispatched to the hour queue, 3 and 4 the day queue and 5 the week queue.

**Task specification**
The config file overwrites the default starting cpus, memory and time for different processes (tasks) - processes are classified by labels denoting broad levels of computational usage. All processes in eager are marked as belonging to one of these labels, and inherit that label's resource configuration.

**NHM HPC singularity cache**
Nf-core pipelines can source the software for each process from either conda, singularity or docker. We strongly recommend using singularity, as this is installed on the NHM HPC and is more robust than conda. To avoid nextflow downloading singularity images for every different process for every different user, we've set up a central singularity cache that can be read by all users. Using the NHM.config file (see the configs page) automatically points nextflow to this cache location. Note that not everyone can write to the cache: if in the rare instance you run a pipeline that includes processes and singularity images not already cached, you may get an error. In this case, please contact us.

## This is hopefully the answer! 

I now need to specify the singularity cache as in the ampliseq example above to run my wf-16S nextflow pipeline on the HPC server.


Let's try this for the demo data, i took this from the ampliseq example above!

```
~/nextflow run epi2me-labs/wf-16s \
--fastq ~/wf-16s-demo/test_data/ \
-c /mbl/share/workspaces/groups/nextflow/config/NHM.config \
-profile genericcpuhour,singularity


#The -c species the config file to use
#The -profile file is a specific set of configuration attributes that can be selected during pipeline execution. With this we specify to use the singularity and which cpu to use: day, hour, etc.
```

~/nextflow run epi2me-labs/wf-16s \
--fastq /mbl/share/workspaces/groups/jungblutlab/Paddy_Nextflow/wf-16s-demo/test_data/ \
-c /mbl/share/workspaces/groups/nextflow/config/NHM.config \
-profile genericcpuhour,singularity


[More info on config files](https://www.nextflow.io/docs/latest/config.html)



ERROR ~ Error executing process > 'fastcat (2)'

Caused by:
  Failed to pull singularity image
  command: singularity pull  --name ontresearch-wf-common-sha91452ece4f647f62b32dac3a614635a6f0d7f8b5.img.pulling.1705923693001 docker://ontresearch/wf-common:sha91452ece4f647f62b32dac3a614635a6f0d7f8b5 > /dev/null
  status : 255
  message:
    INFO:    Converting OCI blobs to SIF format
    INFO:    Starting build...
    Getting image source signatures
    Copying blob sha256:edaedc954fb53f42a7754a6e2d1b57f091bc9b11063bc445c2e325ea448f8f68
    Copying blob sha256:0456c71d8011447360dd461b49b6a1a4cb36edf8f7250817e99ceeb0d353f06b
    Copying blob sha256:38cad909cfed9fb07cc2c6612612435d52302b48ebcf94a2a4b77e02b8414142
    Copying blob sha256:88397e6fd840152eca6643094559385065a6ab98a1c3c7e75eb676b9d97aad97
    Copying blob sha256:e8b63ef4ca0ca489bb848ecf9f4db232528d222a2c0bb2a599b312ce6b5b551a
    Copying blob sha256:f9432144bf11a2ae3ffb7360b4e32f7b18c75c7c8c125274ec3cf49d0de28e24
    Copying blob sha256:cd889bc730171a0013130be60f28f869d7e7cc4d923396628660de9b67f7eec8
    Copying blob sha256:4f4fb700ef54461cfa02571ae0db9a0dc1e0cdb5577484a6d75e68dc38e8acc1
    Copying blob sha256:2042b0824681af825bf6a3c7d48cf91a218427db24a26198aaa7a2e92145081d
    Copying blob sha256:4f4fb700ef54461cfa02571ae0db9a0dc1e0cdb5577484a6d75e68dc38e8acc1
    Copying blob sha256:2f9c1755358372eab764e7d312533e730b38371083030b95b875cd919d5be290
    Copying blob sha256:ea8e698bd9cdd662700e4176c7e1e7a4a0a8edbbb38621cff26f3f74e8b54284
    Copying blob sha256:ec3fa796154c412462e1c3d62868e435f15dd0db49a4d69241b8664b571f5ee1
    Copying blob sha256:4f4fb700ef54461cfa02571ae0db9a0dc1e0cdb5577484a6d75e68dc38e8acc1
    Copying blob sha256:86f85e8133a305a13c0394c2da2eb1af7a6832c286f790b1ddabba02cdb1c844
    Copying blob sha256:d81013941655b8f1ff425dba694e5afb5ea50c874dfdf0f0762043e1dd846ad5
    Copying blob sha256:9d46342db4895c64ef443cc7f166b52d76ad35b580d2bd57d947b1f28a455353
    Copying blob sha256:990735f5fd01c6ed69a585c1ef171beea4d90c40ac3ef2b853c4c13b0b30a5fe
    Copying blob sha256:e1b594c348607c6386d37b0ddccb3de64be2db17d54e41f31a4ee64a43e3a9ff
    Copying blob sha256:4f4fb700ef54461cfa02571ae0db9a0dc1e0cdb5577484a6d75e68dc38e8acc1
    Copying config sha256:26919dbf87dc4266db7524f4e1b39da4bb0ed09808626fd7eafb166fc83216a8
    Writing manifest to image destination
    Storing signatures
    FATAL:   While making image from oci registry: error fetching image to cache: while building SIF from layers: conveyor failed to get: no descriptor found for reference "fdbae72a80c03fcfeb5941c1bc636fbd76afa759d75c0f8d69929738bc5faa22"



## Nextflow parameter issues
According to Thomas, the NHM.config sets up a singularity cache directory, the aim being that users don't need to download all of the singularity images themselves. It seems the permissions for this cache don't have global write access, so as i'm using a pipeline that doesn't already have the images cached, it is not allowing me to write anything in this directory.

To get around this I need to create another config file, which specifies my directory

1. make a directory cache:

`mk dir ~/singularity_cache

2. make a config file in nano and copy in this text:

`nano my.config

Copy this: 

`singularity.libraryDir = '/mbl/share/workspaces/users/patrh5/singularity_cache'

3. Add a second -c argument to the nextflow command

```
~/nextflow run epi2me-labs/wf-16s \
--fastq ~/wf-16s-demo/test_data/ \
-c /mbl/share/workspaces/users/patrh5/my.config \
-c /mbl/share/workspaces/groups/nextflow/config/NHM.config \
-profile genericcpuhour,singularity
```


WARN: Singularity cache directory has not been defined -- Remote image will be stored in the path: /gpfs/nhmfsa/bulk/share/data/mbl/share/workspaces/users/patrh5/work/singularity

-- Use the environment variable NXF_SINGULARITY_CACHEDIR to specify a different location

ERROR ~ Error executing process > 'fastcat (3)'

Caused by:
  Failed to pull singularity image command: singularity pull  --name ontresearch-wf-common-sha91452ece4f647f62b32dac3a614635a6f0d7f8b5.img.pulling.1705935977942 docker://ontresearch/wf-co        mmon:sha91452ece4f647f62b32dac3a614635a6f0d7f8b5> /dev/null status : 255


# Final attemps

This time, I copied the 'NHM.config' to my home directory and edited the file in nano. Not neat code but just wanted to be safe.

```
mv /mbl/share/workspaces/groups/nextflow/NHM.config ~
cd
ls
mv NHM.config PH_NHM.config
```

I then opened the file in nano...

`nano PH_NHM.config

and changed the following lines from:

```
singularity.cacheDir = '/mbl/share/workspaces/groups/nextflow/cache/singularity'
singularity.libraryDir = '/mbl/share/groups/nextflow/cache/singularity>
conda.cacheDir = '/mbl/share/workspaces/users/patrh5/cache/conda'
```

To:

```
singularity.cacheDir = '/mbl/share/workspaces/users/patrh5/nextflow_cache/singularity'
singularity.libraryDir = '/mbl/share/workspaces/users/patrh5/nextflow_cache/singularity'
conda.cacheDir = '/mbl/share/workspaces/users/patrh5/nextflow_cache/conda'
```

This will hopefully reroute the singularity library and cache to my home directory which should be writeable by my nextflow pipelines. 

The rest of the .config file tells it to use slurm and send the files to the CPU nodes. 

Let's retry now with our new config file:

```
~/nextflow run epi2me-labs/wf-16s \
--fastq ~/wf-16s-demo/test_data/ \
-c /mbl/share/workspaces/users/patrh5/PH_NHM.config \
-profile genericcpuhour,singularity
```


OMG, it worked!!

Let's think now about how we can use this for our own data...

I think the first step does the concatenation so i don't think we need this..


The test data input is the raw fastq output from the nanopore run, this means they are still not concatenated:

barcode01/
barcode02/
barcode03/
unclassified/

I have already concatenated my data, but maybe I can use this or maybe i just use the raw fastq pass files, however I have it in my home directory in my directory:

`nut_exp/16S/ont2/

Time to read the manual!!

If i can get to taxonomic annotation, i can then try emu to group them?

# Using wf-16S for my samples

[16S and 18S metabarcoding workflow](https://github.com/epi2me-labs/wf-16s)

- Sequence quality
- Size filtering, adapter removal
- Taxonomic assignment


This workflow can be used for default or custom databases. The default databases are the NCBI targeted loci: 16S rDNA, 18S rDNA, 28S rDNA, and ITS rDNA. 

Taxonomic assignment can be through:

1. minimap2 (alignment based) = default option
2. kraken2 (k-mer based) 

K-mers refers to all the possible substrings of length k that are possible in a string of nucleotides, e.g. a dinucleotide is a kmer where k=2.

When we talk about all kmers to talk about all the possible sequences of that length. So for example, when K=2 all the possible kmers are: AA AT AC AG TA TT TC TG CA CT CC CG GA GT GC GG

K is usually bigger than 2, so we can talk about all 4mers (256 of them), all 6mers (4096 of them), all 7mers (16,384 of them) etc.

Output:
1. An abundance table with counts per taxa in each sample
2. An interactive sankey plot and sunburts plots to explore the different identified lineages.
3. A bar plot comparing the abundances of the most abundant taxa in each sample.

## Install

I have already completed install above, briefly:

1. Install nextflow to manage compute and software resources. 
2. The nextflow pipeline also requires Docker or Singularity to run. We don't have to write a specific singularity / docker image but we need access to the software as the Nextflow pipeline automates writing of the pipeline. We can't easily install Docker on the HPC, however Singularity is available for Nextflow users on the HPC. This requires a slight adjustment, as above.
3. We then specify our nextflow config file, thanks to info from Thomas Creedy (NHM) with the `-profile` argument on our nextflow run.
4. We can then pull the repository into the asset folder of our nextflow install and provide a list of all parameters that are available for the workflow:

`nextflow run epi2me-labs/wf-16S --help`

5. A demodataset is also available for install, do this with the `wget` command:

`wget https://ont-exd-int-s3-euwst1-epi2me-labs.s3.amazonaws.com/wf-16s/wf-16s-demo.tar.gz`
`tar -xzvf wf-16s-demo.tar.gz`

## Running the demo

All nexflow commands use the `nextflow run <pipeline_name>` command

To run the demo dataset on the NHM server we first need to copy across the Nextflow config file created by Thomas Creedy to our home directory, where we have write access. We need write access to house the singularity cache somewhere writeable for our nextflow pipeline.

`mkdir /~/cache/singularity /~/cache/conda

We then rename our copied config file:

`mv ~/NHM.config ~/PH_NHM.config

We can then rewrite our cache and library pathways to our personal cache directory in our home directory

`nano PH_NHM.config

Write the following correction:


Once we have corrected our NHM config file, we can run the nextflow pipeline to analyse the test data using:

```
~/nextflow run epi2me-labs/wf-16s \
--fastq ~/wf-16s-demo/test_data/ \
-c /mbl/share/workspaces/users/patrh5/PH_NHM.config \
-profile genericcpuhour,singularity

#-fastq specifies which files to run the pipeline on
#-c specifies where to find our config file
#-profile this specifies which profile inside our config file to use, genericcpuhour tells it to use the HPC-CPU-node for 1 hour, and tells nextflow to use singularity, not docker.
```

[troubleshooting](https://labs.epi2me.io/wfquickstart/)


## Inputting my DNA

### Input options

Nextflow parameter name | Type | Description | Help | Default |
|--- | --- | --- | --- | --- |
|fastq|string|FASTQ files to analyze|This accepts one of three cases (i) path to the single FASTQ file, (ii) the path to a top-level directory containing FASTQ files, (iii) path to a directory containing one level of sub-directories which contain FASTQ files, in the first and second case, a sample name can be supplied with --sample, in the third case, the data is assumed to be multiplexed with the names of the sub-directories a barcodes. In this case a sample sheet can be provioded with --sample sheet | |
|classifier|strig|Kraken" or Minimap2 workflow|Use Kraken2 for fast classification and minimap2 for finer resolution|minimap2|
|analyse_unclassified|boolean|Analyse unclassified reads from input directory. By default the workflow will not process reads in the unclassified directory.|If selected and the input is a multiplex directory the workflow will also process the unclassified directory|False|
|exclude host|string|A FASTA or MMI file of the host reference. Reads that align will be excluded from the analysis| | |


### Sample Options

Nextflow parameter name | Type | Description | Help | Default |
|---| ---|---|---|---|
| sample_sheet | string | A .csv file used to map barcodes to sample aliases. The sample sheet can be provided when the input data is a directory containing sub-directories with FASTQ files.| The sample sheet is a .csv file, with columns named barcode, sample_id, alias. Extra columns are allowed. A type column is required for certain workflows and should have the following values: test_sample, positive_control, negative_control, no_template_control.| |
| sample | string| a single sample name for non-multiplexed data. Permissible if passing a single .fastq(.gz) file or directory of .fastq(.gz) files.|||

I need to make a sample sheet for the original barcode files, not my concatenated versions!

## Reference options

Here is how we specify the database we want to use for our analysis:

*It looks like you have a database and a taxonomy, not quite sure!*

Nextflow parameter name | Type | Description | Help | Default |
|---|---|---|---|---|
| database_set | string | sets the reference, databases and taxonomy datasets that will be used for classifying reads. | This setting is overridable by providing an explicit taxonomy, database, or reference path **in the other reference options** | ncbi_16S_18S |
| database | string | not required but can be used specifically override Kraken2 database, either with a .tar.gz file or Directory | By default uses database chosen in database_set parameter ||
| taxonomy | string | not required but can be used to specifically override taxonomy database. Change the default to use a different taxonomy file, either .tar.gz or directory | By default NCBI taxonomy file will be downloaded and used. | |
| reference | string | override the FASTA reference file selected by the database_set parameter. It can be a FASTA format reference sequence collection or minimap MMI format index | This option should be used in conjunction with the database parameter to specify a custom database | |
| ref2taxid | string | not required but can be used to specify a ref2taxid mapping, format is a .tsv (refname taxid), no header row | by default uses ref2taxid for option chosen in database_set parameter. | |
| taxonomic_rank | string | returns results at the taxonomic rank chosen. In kraken2 pipeline: set the level that Bracken will estimate abundance. at Default: G (genus). For species specify S | | G |

## Kraken2 Options

Nextflow parameter name | Type | Description | Help | Default |
|---|---|---|---|---|
|bracken_length | integer | set the length value for Bracken | Should be set to the length used to generate the kmer distribution file supplied in the Kraken database input directory. For the default datasets these will be set automatically. ncbi_16s_18s = 1000 , ncbi_16s_18s_28s_ITS = 1000 , PlusPF-8 = 300 | |
|kracken2_memory_mapping | boolean | avoids loading database into RAM | Kraken 2 will by default load the database into process-local RAM; this flag will avoid doing so. It may be useful if the available RAM memory is lower than the size of the chosen database. | True |
|include_kraken2_assignments|boolean|A per sample TSV file that indicates how each input sequence was classified as well as the taxon that has been assigned to each read. The TSV's will only be output on completion of the workflow||
|kraken2_confidence|number|Kraken2 confidence score theshold, default valid: 0.0, interval: 0-1. | Apply a threshold to determine if a sequence is classified or unclassified. Please visit the following link for further details about how it works: https://github.com/DerrickWood/kraken2/wiki/Manual#confidence-scoring. | 0.0|

## Report Options

Nextflow parameter name | Type | Description | Help | Default |
|---|---|---|---|---|
| abundance_threshold | number | remove those taxa whose abundance is equal or lower than the chosen value | remove taxa with abundances lower than or equal to a relative value, use a decimal between 0-1. To remove taxa to an absolute value, use larger than 1 | 1 |
| n_taxa_barplot | integer | number of most abundance taxa to be displayed in the barplot, the rest of the taxa will be grouped in "other" | |8|

## Advanced Options

Nextflow parameter name | Type | Description | Help | Default |
|---|---|---|---|---|
| min_len | integer | specify read length lower limit | any reads shorter than this will be removed | 800 |
| min_read_qual | number | specify read quality lower limit | any reads with a quality lower than this limit will be not included |
| max_len | integer | specify read length upper limit | any reads longer than this limit will not be included in the analysis | 2000 |
| threads | integer | number of CPU threads per workflow task | several tasks benefit from multiple CPU threads. This option sets the number of CPU threads. The total CPU resource is constrained by the executor configuration. | 4 |

## Outputs

|Title | File path | Description|
|---|---|---|
|Workflow report | ./wf-metagenomics-report.html | report for all samples |
|Abundance table with counts per taxa | ./abundance_table_<taxonomic_rank>.tsv | Per-taxa counts TSV, including all samples |
| [Bracken report file](https://github.com/jenniferlu717/Bracken#output-kraken-style-bracken-report) | ./bracken/<alias>,kraken2_bracken.report | TSV file with the abundance of each taxa. |
| [Keaken2 taxonomic assignment per read] | ./kraken2/<alias>.kraken2.report.txt | Lineage-aggregated counts. |
| Kraken2 taxonomic assignment per read | ./kraken2/<alias>.kraken2.assingments.tsv|TSV file with the taxonomic assignment per read. |
| BAM files | ./host_bam/<alias> | Varius BAM files generated from filtered input reads to reference files |


# wf-16S nextflow pipeline overview

## 1. concatenate input files 

Fastcat is used to concatenate input FASTQ files prior to downstream processing of the workflow. 

It will also output per-read stats including read lengths and average qualities. 

You may choose to filter reads using:
- min_len
- max_len
- min_read_qual

## 2. remove host sequences

We have included an optional filtering step to remove any host sequences that map against a provided host reference using minimap2. This can be used for removing human reads.

To do this, provide the path to your host reference with the exclude_host parameter. The mapped reads are output into a BAM file and excluded from further analysis.

## 3. Classify reads taxonomically 

We can use Minimap2 or Kraken2 

### Minimap2

Minimap2 provides better resolution, but can take significantly more time. 

`nextflow run epi2me-labs/wf-16s --fastq test_data/case01 --classifier minimap2`

### Kraken2

Kraken2 provides the fastest method for taxonomic classification. Supports real-time analysis.

Then, Bracken is used to provide an estimate of the species / genus abundance in the sample.

## Build a custom database for analysis

How can I use custom databases? - If you want to run the workflow using your own Kraken2 database, you'll need to provide the database and an associated taxonomy dump. For a custom Minimap2 reference database, you'll need to provide a reference FASTA (or MMI) and an associated ref2taxid file. For a guide on how to build and use custom databases, take a look at our article on how to run wf-16s offline:

https://labs.epi2me.io/how-to-meta-offline/

How to select the correct databases:
https://labs.epi2me.io/metagenomic-databases/


4. Prepare output

The main output of the wf-16S pipeline is the `wf-16s-report.html`. Which can be found in the output directory. It contains a summary of read statistics, the taxonomic composition of the sample and some diversity metrics. The results shown in the report also can be customized, with the report option arguments listed above.

The workflow output also contains Kraken and bracken reports for each sample. 

Additionally, the ‘species-abundance.tsv’ is a table with the counts of the different taxa per sample. 

You can use the flag `include_kraken2_assignments` to include a per sample TSV file that indicates how each input sequence was classified as well as the taxon that has been assigned to each read. This TSV file will only be output on completion of the workflow.

# Troubleshooting 

-[common nextflow exit codes](https://labs.epi2me.io/trouble-shooting/)
-When using the Minimap2 pipeline with a custom database, you must make sure that the ref2taxid and reference files are coherent, as well as the taxonomy database.
-If your device doesn't have the resources to use large Kraken2 databases, you can enable kraken2_memory_mapping to reduce the amount of memory required.






#Trying wf-16S for my data

To start with I am going to try this approach on two of my 16S samples.

I need a script that:

1. Concatenates the barcode and applies an 'alias' sample name using a sample_sheet.csv
2. Filters reads to remove anything 1000 < x < 2000  
3. Removes adapters
4. Uses the Kraken2 database for taxonomic analysis 


trim barcodes was on, on the Minknow software, do i still need to trim adapters?




```
~/nextflow run epi2me-labs/wf-16s \
--fastq nut_exp/16S/ont2/ \
-c /mbl/share/workspaces/users/patrh5/PH_NHM.config \
-profile genericcpuhour,singularity \
--classifier kraken2 \
--sample_sheet ~/nut_exp/my_data_test.csv \
--database_set ncbi_16s_18s \
--taxonomic_rank G \
--include_kraken2_assignments true \
--out_dir ~/nut_exp/test_output \
--min_len 1000 \
--max_len 2000 \
--min_read_qual 9
```
Need to specify output file

Let's try a simpler one:

```
~/nextflow run epi2me-labs/wf-16s \
--fastq nut_exp/16S/ont2/ \
-c /mbl/share/workspaces/users/patrh5/PH_NHM.config \
-profile genericcpuhour,singularity \
--sample_sheet ~/nut_exp/my_data_test.csv \
--database_set ncbi_16s_18s
```


seems like issue might have been with Kraken

For now let's just set off a screen containing all 16S with minimap to see what happens, we can always do other stuff in the mean time

I think it would be best to do this with genericcpuday this time, as it might take a while.

I am going to add the max and min length and min read quality requirements which might improve speed

`screen -S 16S_minimap

```
~/nextflow run epi2me-labs/wf-16s \
--fastq nut_exp/16S/ont2/ \
-c /mbl/share/workspaces/users/patrh5/PH_NHM.config \
-profile genericcpuday,singularity \
--sample_sheet ~/nut_exp/16S_sample_sheet.csv \
--database_set ncbi_16s_18s \
--min_len 1000 \
--max_len 2000 \
--min_read_qual 9 

```


# 18S analysis for nutrient experiment
Going to make sample sheets for the 18S samples, I made seperate sample sheets for each sample...

18S ont1

```
~/nextflow run epi2me-labs/wf-16s \
--fastq nut_exp/18S/ont1/ \
-c /mbl/share/workspaces/users/patrh5/PH_NHM.config \
-profile genericcpuday,singularity \
--sample_sheet ~/nut_exp/18S_ont1_sample_sheet.csv \
--database_set ncbi_16s_18s \
--min_len 1000 \
--max_len 2000 \
--min_read_qual 9 \
--out_dir nut_exp/18S_output/ont1_minimap
```

18S ont2

```
~/nextflow run epi2me-labs/wf-16s \
--fastq nut_exp/18S/ont2/ \
-c /mbl/share/workspaces/users/patrh5/PH_NHM.config \
-profile genericcpuday,singularity \
--sample_sheet ~/nut_exp/18S_ont2_sample_sheet.csv \
--database_set ncbi_16s_18s \
--min_len 1000 \
--max_len 2000 \
--min_read_qual 9 \
--out_dir nut_exp/18S_output/ont2_minimap
```

Running all my samples with minimap

The adapters have been removed, specific quality score set, and the analysis run for nutrient experiment

Downsides - minimap2 not kraken, 18S database not ideal

Output of minimap2 run:

Look good, not a good coverage for protists, only fungi in 18S

# Running wf-16S with Kraken2

## Ont1 18S

this didn't work - why??

## Ont2 18S

this didn't work - why??

## 16S 

```
~/nextflow run epi2me-labs/wf-16s \
--fastq nut_exp/16S/ont2/ \
-c /mbl/share/workspaces/users/patrh5/PH_NHM.config \
-profile genericcpuday,singularity \
--sample_sheet ~/nut_exp/16S_sample_sheet.csv \
--database_set SILVA_138_1 \
--classifier kraken2 \
--min_len 1000 \
--max_len 2000 \
--min_read_qual 9 \
--out_dir nut_exp/16S_output/kraken2_silva138
```

# Running wf-16S with a custom database




# Emu

If i can't get Nextflow to work, i'm going to try Emu!


I downloaded the PR2 database from here:
https://gitlab.com/treangenlab/emu/-/issues/49

And transferred it to my home directory from my local PC:





# Base quality scores

https://www.metagenomics.wiki/pdf/qc/phred-base-quality

Quality value for each sequenced base
The Phred score is a measure for base quality in DNA sequencing. The larger the Phred value, the better the quality of a sequenced base.

|Phred|Error-rate|Accuracy|
|---|---|---|
|10|1 of 10|90%|# very low quality|
|20|1 of 100|99%|# minimum quality for many tools|
|30|1 of 1000|99.9%|# reasonable good quality|
|40|1 of 10000|99.99%|# high quality|
|50|1 of 100000|99.999%|# very high quality|

