# ONT full length 16S and 18S rRNA gene markdown sequencing

This Markdown document was created by Patrick M. Hooper on 12/02/2024.

# Preamble

This document provides the process I used for analysis of my raw ONT data. The document covers:

1. Offloading data from a GridION / MinKNOW run
2. Concatenating fastq.gz files
3. Filtering reads
4. Taxonomic classification with [Emu](https://gitlab.com/treangenlab/emu)

## Assumptions

Firstly, this markdown document assumes you have copied your ONT sequencing run from the NHM GridION onto your group directory on the HPC. For the jungblut group this is /mbl/share/workspaces/groups/jungblutlab/

***Very Important Sidenote ***
*I copied my sequencing data into the jungblutlab from the GridION using scp into my folder 'Paddy_Nanopore'. Within this, you can then find the two sequencing runs, ONT_1_18S (which contains the 18S data for the environmental microbiomes chapter (except soil, deep mats and WHIS)) and ONT_2 (which contains the nutrient experiment data, soil, deep mat biofilms, and WHIS1).*

***These folders should never be modified!!***

Secondly, it is worthwile making a metadata file for your samples now for correct renaming of your ONT barcodes (i.e. barcode01-barcode96) to your sample name (e.g. WHL_M1).

Thirdly, this markdown document mainly uses SBATCH scripts designed for submitting jobs to the NHM HPC servers. If you are unfamiliar with the NHM HPC, please read 'NHM_HPC_slurm.md'.

Fourthly, you will want to use ['screen'](https://linuxize.com/post/how-to-use-linux-screen/) for some of your bigger processing steps, this gives you the possibility of making a separate temporary terminal which you can then 'attach' and 'detach' from, meaning you can send a command, detach, and close your terminal, all while the command still runs in your screen. When you think it's done, you just re-attach to the screen to continue working. 

Finally, all software in this pipeline is installed through conda, please see the NHM HPC user guide for info on installing miniconda and for further info, please read my markdown file 'Conda_for_NHM_HPC'.

# 1. Summary 
- This markdown file provides a summary of the initial processing, quality control, quality filtering, and taxonomic annotation of my 16S and 18S rRNA gene metabarcoding sequencing data.
- The seqeuncing data was produced using the Native Barcoding Kit 96 V14 (SQK-NBD114.96) on a MinION Flow Cell (R10.4.1; FLO-MIN114).
- The MINKNOW software produces several output files. The data I worked on was contained in the 'fastq_pass' folder. 
- The DNA data was basecalled and demultiplexed by GuPPY in the MinKNOW software as part of the sequencing process and therefore my final sequencing data was provided in separate barcode subdirectories for each sample. Within each barcode subdirectory, the .fastq.gz files were split over several folders.

**MinKNOW output**
The sequencing run on the MinKNOW software will produce a set output folder containing:

1. fastq_fail
2. fastq_pass - this is what we will use for our analysis
3. fast5_fail / pod5_fail
4. fast5_pass / pod5_pass - this is the sequencing data without the nucleotide annotation
5. report.html - this gives a summary of the sequencing run
6. report.json
7. report.md
8. barcode alignment file
9. final summary
10. pore activity data
11. sequencing summary
12. throughput summary

___

# 2. Organising my sequencing data into my project directories

## Transferring my data into my home directory 
 Move the .fastq files from your MinKNOW output directory into your home directory, you want to copy the entire directory called 'fastq_pass' into a directory within your home directory called 'ONT_run'. We have to use the -r argument here to copy the entire directory

`cp -r /mbl/share/workspaces/groups/jungblutlab/[your-ONT-directory]/fastq_pass/ ~/ONT_run/

For my data this looked like:

`mkdir ~/ont1 ~/ont2
`cp -r /mbl/share/workspaces/groups/jungblutlab/Paddy_Nanopore/ONT_1_18S/PH_2023-09-06_ARCTIC_18S/full_length_18S_rRNA_gene_reads_run_1/20230906_1808_X4_FAX39760_9alf312d/fastq_pass ~/ont1
`cp -r /mbl/share/workspaces/groups/jungblutlab/Paddy_Nanopore/ONT_2/PH_2023-12-04_Exp_3/Nutrient_Exp_Soil_Mats/20231204_1901_X5_FAV21302_c31230e9/fastq_pass/ ~/ont2

## Creating project directories
My sequencing consisted of two separate ONT runs, ont1 and ont2, which together contained the sequencing data for two of my PhD chapters, but with different samples for the different projects split over the two runs. I therefore decided to make a separate subdirectory and copy across all the barcodes relevant for each project into directories called: 'nut_exp' for the nutrient experiment chapter and 'wh_18S' for the 18S survey of Ward Hunt island chapter.

`mkdir ~/nut_exp ~/wh_18S

Within the nutrient experiment folder we will make a subdirectory for the 16S and 18S reads

`mkdir ~/nut_exp/16S ~/nut_exp/18S

I quickly made sub-folders for my 18S data in the wh_18S and nut_exp/18S directories for ont1 and ont2, to avoid mixing up barcode numbers when I rename them with my sample IDs.

`mkdir ~/nut_exp/18S/ont1 ~/nut_exp/18S/ont2 ~/wh_18S/ont1 ~/wh_18S/ont2

## Copying my run data into my project directories

The whole of ont1 run was for the wh_18S project, therefore I copied all 96 barcode folders across

`cp -r ~/ont1_fastq/fastq_pass/barcode* ~/wh_18S/ont1

The rest of the barcode samples were copied one by one to make sure this was done correctly, based on my metadata table containing information on the sample ID for each barcode across the two runs.

___

# 3. Concatenation of ONT sequencing runs

The data that arrives is barcoded, however, sometimes we might want to look at the average quality of our sequencing data or maybe just a certain subset of samples. Some packages allow you to compare samples by specifying a directory of barcodes or perhaps listing your .fastq files of interest. However, sometimes it requires some coding on our end. One way we could review the quality of our ont run would be to concatenate all the barcode subdirectory files (each containing several .fastq files) into one large .fastq.gz file for each project. 

Let's first do some housekeeping and make a directory in our home directory for storing logs from our HPC slurm jobs. we can specify that job outputs and errors are sent here to keep our home directory tidy.

`mkdir ~/logs

This script should concatenate all .fastq files in our sequencing run folders `ont1_fastq` and `ont2_fastq`. 

First open a new script:

`nano ~/concat_ONT_runs.sh

Then copy the below code into your opened .sh script file:

```
#!/bin/bash
#SBATCH -J combine_ont_run_fastqs
#SBATCH -p hour
#SBATCH --error=~/logs/combine_ont_run_fastqs_err.txt
#SBATCH --outout=~/logs/combine_ont_run_fastqs_out.txt

zcat ~/ont1_fastq/fastq_pass/barcode*/*.fastq.gz | gzip > ~/ont1_fastq/ont1_concat.fastq.gz
echo "ont1_concat.fastq.gz created"

zcat ~/ont2_fastq/fastq_pass/barcode*/*.fastq.gz | gzip > ~/ont2_fastq/ont2_concat.fastq.gz
echo "ont2_concat.fastq.gz created"

```

*Note: We are using zcat here instead of cat. If you concatenate gzip files using cat, you'll end up with multiple headers and footers in the resulting file, which will cause problems when you try to decompress or read the file. Therefore, we use zcat to decompress the data to standard output and then pipe that output to gzip to compress it again into a single file*

To runt this jobs we need to send them to the HPC servers using the `sbatch` command. 

`sbatch ~/concat_ONT_runs.sh

You should get a notification your slurm job has been submitted and a unique ID. You can check your job is running by typing `squeue` in the command line.

The output of this step will be two large files called `ont1_concat.fastq.gz` and `ont2_concat.fastq.gz` in the home directory. We will use these later for analysis of the sequencing run.

___

# 4. Concatenation of sample barcode directories

Now that I have the correct samples in my project folders, I need to create my first bash script to convert my barcode folders (containing several subdirectories of single .fastq files), into one renamed .fastq file.

Let's make a folder for each project

`mkdir ~/nut_exp/single_fastq_barcodes ~/wh_18S/single_fastq_barcodes

## Nutrient Experiment Samples
For each barcode folder, we want to combine all the sub-folders into one single fastq.gz file and rename that file with the correct sample ID. Let's make a script to do this for the nutrient experiment samples first.

`mkdir ~/nut_exp/scripts
`nano ~/nut_exp/scripts/single_fastq_barcodes.sh

This should open a basic text editor, copy the below script into the file you have just opened. We use right hand click to paste.

*Note: The #SBATCH commands are used here to give information to the job scheduler. Here I am using 4 arguments:
*-J = assign a job name
*-p = submit the job to the 'hour' division within slurm, day, week, etc. also available
*--error = send any error messages to a text file as defined by the user
*--output = send my output to a specifically named file of your choice, otherwise it just writes a '.out' file named after the slurm job which quickly gets hard to keep track of.

*Note how for the error and output arguments I am writing the absolute directory, it doesn't work when you use '~' for your home directory 

```
#!/bin/bash
#SBATCH -J nut_exp_sfbs
#SBATCH -p hour
#SBATCH --error="/home/patrh5/logs/nut_exp_sfbs_err.txt"
#SBATCH --output="/home/patrh5/logs/nut_exp_sfbs_out.txt"

zcat /home/patrh5/nut_exp/18S/ont1/barcode10/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_TZ3_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont1/barcode11/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_TZ2_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont1/barcode12/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_TZ1_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont1/barcode49/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_TZ1_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont1/barcode61/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_TZ4_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont1/barcode62/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_TZ3_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont1/barcode64/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_TZ2_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont1/barcode89/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/NC_18S.fastq.gz

echo "ont1 18S created"

zcat /home/patrh5/nut_exp/16S/ont2/barcode01/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_N4_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode02/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_N3_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode03/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_N2_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode04/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_N1_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode05/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_C4_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode06/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_C3_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode07/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_C2_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode08/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_C1_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode09/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_NP4_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode10/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_NP3_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode11/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_NP2_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode12/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_NP1_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode13/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_P4_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode14/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_P3_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode15/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_P2_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode16/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_P1_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode17/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_N4_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode18/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_N3_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode19/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_N2_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode20/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_N1_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode21/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_C4_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode22/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_C3_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode23/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_C2_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode24/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_C1_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode25/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_NP4_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode26/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_NP3_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode27/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_NP2_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode28/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_NP1_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode29/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_P4_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode30/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_P3_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode31/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_P2_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode32/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_P1_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode33/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/NC_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode34/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_TZ3_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode35/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_TZ2_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode36/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_TZ1_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode37/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_TZ4_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode38/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_TZ3_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode39/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_TZ2_16S.fastq.gz
zcat /home/patrh5/nut_exp/16S/ont2/barcode40/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_TZ1_16S.fastq.gz

echo "ont2 16S created"

zcat /home/patrh5/nut_exp/18S/ont2/barcode41/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_N4_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode42/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_N3_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode43/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_N2_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode44/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_N1_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode45/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_C4_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode46/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_C3_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode47/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_C2_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode48/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_C1_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode49/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_NP4_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode50/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_NP3_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode51/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_NP2_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode52/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_NP1_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode53/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_P4_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode54/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_P3_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode55/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_P2_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode56/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_P1_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode57/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_N4_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode58/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_N3_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode59/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_N2_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode60/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_N1_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode61/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_C4_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode62/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_C3_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode63/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_C2_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode64/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_C1_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode65/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_NP4_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode66/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_NP3_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode67/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_NP2_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode68/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_NP1_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode69/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_P4_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode70/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_P3_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode71/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_P2_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode72/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/W_P1_18S.fastq.gz
zcat /home/patrh5/nut_exp/18S/ont2/barcode93/*.fastq.gz | gzip > /home/patrh5/nut_exp/single_fastq_barcodes/M_TZ1_18Sb.fastq.gz

echo "ont2 18S created"
```

## Ward Hunt Environmental Biomes Samples

Let's make a script to do this for the Ward Hunt environmental biomes project 18S samples. These samples are split over ont1 and ont2.

`mkdir ~/wh_18S/scripts
`nano ~/wh_18S/scripts/single_fastq_barcodes.sh

This should open a basic text editor, copy the below script into the file you have just opened. We use right hand click to paste.

```
#!/bin/bash
#SBATCH -J wh_18S_sfbs
#SBATCH -p hour
#SBATCH -e="/home/patrh5/logs/wh_18S_sfbs_err.txt"
#SBATCH -o="/home/patrh5/logs/wh_18S_sfbs_out.txt"

zcat /home/patrh5/wh_18S/ont1/barcode01/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_W2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode02/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_W1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode03/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_W3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode04/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_W2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode05/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_W1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode06/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_W3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode07/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_W2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode08/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_W1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode09/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W4.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode10/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode11/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode12/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode13/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/AP_W3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode14/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_W3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode15/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/AP_W2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode16/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/AP_W1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode17/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W12.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode18/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W11.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode19/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W10.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode20/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W9.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode21/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W8.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode22/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W7.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode23/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W6.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode24/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_W5.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode25/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_M1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode26/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/AP_M7.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode27/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/AP_M6.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode28/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/AP_M5.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode29/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/AP_M4.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode30/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/AP_M3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode31/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/AP_M2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode32/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/AP_M1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode33/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_M4.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode34/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_M3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode35/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_M2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode36/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_M1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode37/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_M5.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode38/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_M4.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode39/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_M3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode40/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_M2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode41/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WHIS2_M1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode42/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WHIS1_M3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode43/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_M5.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode44/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_M4.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode45/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_M3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode46/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_M2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode47/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_M1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode48/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_M5.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode49/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_M3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode50/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_M2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode51/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_M1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode52/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WHIS3_M3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode53/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WHIS3_M2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode54/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WHIS3_M1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode55/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WHIS2_M3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode56/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WHIS2_M2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode57/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_M4.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode58/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_M3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode59/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_M2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode60/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_M1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode61/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_M7.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode62/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_M6.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode63/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_M5.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode64/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_M4.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode65/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_DM3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode66/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_DM2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode67/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_M10.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode68/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_M9.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode69/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_M8.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode70/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_M7.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode71/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_M6.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode72/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/QL_M5.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode73/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_C5.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode74/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_C4.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode75/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_C3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode76/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_C2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode77/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_C1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode78/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/LA_M3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode79/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/LA_M2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode80/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/LA_M1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode81/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_C3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode82/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_C2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode83/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_C1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode84/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_C5.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode85/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_C4.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode86/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_C3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode87/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_C2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode88/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_C1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode89/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/NC.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode90/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_S5.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode91/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_S3.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode92/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_S2.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode93/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_S1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode94/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_S1.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode95/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_C5.fastq.gz
zcat /home/patrh5/wh_18S/ont1/barcode96/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_C4.fastq.gz

echo "single file barcodes created from ont1!"

zcat /home/patrh5/wh_18S/ont2/barcode73/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_S3.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode74/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_S2.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode75/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_S1b.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode76/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_S5.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode77/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_S4.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode78/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_S3.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode79/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_S2.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode80/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT1_S1.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode81/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WHIS1_M2.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode82/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WHIS1_M1.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode83/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_S4.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode84/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_S3b.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode85/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_S2b.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode86/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT3_S1b.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode87/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_S5.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode88/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WT2_S4.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode89/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_DM3b.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode90/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_DM2b.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode91/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_DM1.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode92/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_M5b.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode93/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_M3b.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode94/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_M2b.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode95/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WH_M1b.fastq.gz
zcat /home/patrh5/wh_18S/ont2/barcode96/*.fastq.gz | gzip > /home/patrh5/wh_18S/single_fastq_barcodes/WHIS_M3b.fastq.gz

echo "single file barcodes created from ont2!"

```

To submit these jobs we need to send them to the HPC servers using the `sbatch` command. 

`sbatch ~/nut_exp/scripts/single_fastq_barcodes.sh
`sbatch ~/wh_18S/scripts/single_fastq_barcodes.sh

___

# 5. NanoPlot analysis of my two ONT sequencing runs, ont1 and ont2

The first step is to produce some summary data for how well my sequencing runs went overall. To do this we will use a package called [NanoPlot](https://github.com/wdecoster/NanoPlot) . NanoPlot is part of the suite of ONT analysis software developed by [Wouter De Coster](https://github.com/wdecoster) called [NanoPack](https://github.com/wdecoster/nanopack?tab=readme-ov-file) .

[NanoPack reference](https://academic.oup.com/bioinformatics/article/39/5/btad311/7160911?login=false)

## Install NanoPlot

Let's create a conda / mamba environment to house the various packages within NanoPack we are going to use. 

```
mamba create -n nanopack-env -c bioconda nanoplot
conda activate nanopack-env
python --version
conda list 

#This block of code is creating a new conda environment 'nanopack-env' and placing an install of nanoplot inside. 
#We then activate this environment, note how we use mamba for install as it is generally quicker, but conda for activation and deactivation of environments. 
#The python scripts are written and tested for Python >= 3.6, so we check this with --version
#conda list should now show all the packages in our nanopack-env which should include 'nanoplot' and python
```

NanoPlot will produce:
1. Statistical summary of our sequence data
2. Plots of read length, quality, etc.
3. A html summary file that contains all output results (similar to Epi2me without the analysis)

## Using NanoPlot to summarize our sequencing runs

The first thing we want to do is see how succesful our sequencing efforts were. NanoPlot allows us to create statistics on our sequence data quality and average lengths and plots of our sequence quality distribution. This is a great place to start in understanding how well our amplicon sequencing went.

What i'm looking for in this data:

1. Quality scores for our sequencing data, this tells us how reliable our sequencing efforts were
2. The average read length, this is very important for amplicon data as it will tell us if our desired metabarcoding amplicon regions are the main sequenced genetic material. For this we need to know the expected size of our amplicons
3. The analysis of how are sequencing run went over the 72 hour run time, less essential but useful to know. 

## NanoPlot Usage

**File Formats**

NanoPlot takes data in the following formats:
- fastq files(can be bgzip, bzip2 or gzip compressed)
- fastq files generated by albacore, guppy or MinKNOW containing additional information (can be bgzip, bzip2 or gzip compressed)
- sorted bam files
- sequencing_summary.txt output table generated by albacore, guppy or MinKnow basecalling (can be gzip, bz2, zip and xz compressed)
- fasta files (can be bgzip, bzip2 or gzip compressed)
- Multiple files of the same type can be offered simultaneously

Specified using these arguments:

`--summary file #argument to use for summary files generated by guppy
`--fastq_rich #Data is in one or more fastq file(s) generated by albacore, MinKNOW or guppy with additional information concerning channel and time.
`--fastq #Data is in one or more default fastq file(s).

For our work, we will use concatenated ONT files which using `--fastq_rich` as our data contains information about channels and time which is useful to have in our summary report.

**NanoPlot Arguments**

|Argument|Function|
|---|---|
|--raw|store extracted data in tab separated file|
|--huge|input data is one very large file|
|-o, --outdir|specify directory in which output is to be created|
|--tsv_stats|output stats as a properly formatted TSV|

Filtering and Transformation
|Argument|Function|
|---|---|
|--maxlength|Hide reads longer than length specified|
|--minlength|Hide reads shorter than length specified|
|--drop_outliers|Drop outlier reads with extreme long length|
|--downsample|Reduce dataset to N reads by random sampling|
|--loglength|Additonally show logarithmic scaling of lengths|
|--minqual|Drop reads with an average quality score lower than number you specify|

Customization
|Argument|Function|
|-c, --color|speciy a valid matplotlib color for the plots|
|-cm, --colormap|specify a valid matplotlib colormap for the heatmap|
|-f, --format|Specify the output format of the plots, which are in addition to the html files. Arguments: png,jpg,jpeg,webp,svg,pdf,eps,json|
|--listcolors|list the colors which are available for plotting and exit|
|--listcolormaps|list the colors available for plotting and exit|
|--N50|Show the N50 mark in the read length histogram|

**Analyzing our sequencing runs ont1 and ont2**

To produce a summary for our two sequencing runs, we will use the two large concatenated files we produced earlier in Section 3. 

We input our two concat files: 'ont1_concat.fastq.gz' and 'ont2_concat.fastq.gz' into NanoPlot. 

**NOTE: To use NanoPlot you will need your NanoPack-Env we created earlier activated. 'nanopack-env' should appear next to 'your-username@hpc-head-002:', if not you need to activate this conda environment using `conda activate nanopack-env`**

Make a script file and make directories for the output files in your fasta file folders, there are a lot of files

`mkdir ~/ont1_fastq/nanoplot_output/ ~/ont2_fastq/nanoplot_output/
`nano ~/nanoplot_ONT_runs.sh

Copy in the below script:

```
#!/bin/bash
#SBATCH -J ONT_run_analysis_nanoplot
#SBATCH -p hour
#SBATCH --error="/home/patrh5/logs/ONT_run_analysis_nanoplot_err.txt"
#SBATCH --output="/home/patrh5/logs/ONT_run_analysis_nanoplot_out.txt"

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the nanopack-env and specifiy nanoplot version
conda activate nanopack-env

NanoPlot --fastq_rich /home/patrh5/ont1_fastq/ont1_concat.fastq.gz --huge --verbose --tsv_stats -o /home/patrh5/ont1_fastq/nanoplot_output/
echo "ONT 1 report finished"
NanoPlot --fastq_rich /home/patrh5/ont2_fastq/ont2_concat.fastq.gz --huge --verbose --tsv_stats -o /home/patrh5/ont2_fastq/nanoplot_output/
echo "ONT 2 report finished"

#Note how I specified my conda environment within my script, so that there are no complications when it is submitted to the job scheduler
#This script will output the files to the output directories. The --huge command specifies we are handling a large fastq file.
#Note the command is NanoPlot, not Nanoplot or nanoplot
#--fastq_rich specifies my .fastq files were produced by MinKNOW which contain information about the sequencing run as well as the sequencing data
#--huge specifies that this is a particularly large .fastq file
#--verbose tells the script to write out what it is processing to my output file
#--tsv_stats also outputs a .tsv file containing the statistic
#-o specifies a directory for the outputted files

```

Run this script by sending it to the job scheduler:

`sbatch ~/nanoplot_ONT_runs.sh

The output of this will be several .png static images and .html images. There will also be a html report which contains all the information summarized in one tab. In this report I can find the answer to my above questions about how good my two sequencing runs were!

# 6. Nanoplot analysis of thesis chapter sequencing data

Now that we have reviewed the ONT runs overall, we are going to assess the quality of my project specific datasets. It would be interesting to see for instance, how the quality compares between the 16S and 18S data in the nutrient experiment. To do this, we need to concatenate these different groups of samples.

As we have already concatenated our samples for each project into single fastq files, we can use zcat on these files to make our total concatenated files. I will do this for my nutrient experiment and ward hunt environmental microbiome projects. 

`mkdir ~nut_exp/concat_fastqs ~/wh_18S/concat_fastqs

## Concat Nutrient Experiment files

Concatenated 16S and 18S reads including their respective negative control samples:

`nano /home/patrh5/nut_exp/scripts/concat_raw_16S_18S_fastqs.sh

Copy this code into this .sh file:

```
#!/bin/bash
#SBATCH -J zcat_nut_exp
#SBATCH -p hour
#SBATCH --error="/home/patrh5/logs/zcat_nut_exp_err.txt"
#SBATCH --output="/home/patrh5/logs/zcat_nut_exp_out.txt"

#This script runs through zcat to create two large .fastq files, containing the 16S and 18S rRNA genes, respectively. This large file will form the input data for Nanoplot.

#16S single fastq barcodes
echo "zcat all 16S files"
zcat /home/patrh5/nut_exp/single_fastq_barcodes/*16S.fastq.gz | gzip > /home/patrh5/nut_exp/concat_fastqs/concat_raw_16S.fastq.gz
echo "16S file completed"

#18S single fastq barcodes
echo "zcat all 18S files"
zcat /home/patrh5/nut_exp/single_fastq_barcodes/*18S.fastq.gz | gzip > /home/patrh5/nut_exp/concat_fastqs/concat_raw_18S.fastq.gz
echo "18S file completed"
```

run the script:
`sbatch /home/patrh5/nut_exp/scripts/concat_raw_16S_18S_fastqs.sh

## Concat Ward Hunt Microbiomes files

Total 18S rRNA samples with negative control:

`nano /home/patrh5/wh_18S/scripts/concat_raw_18S_fastqs.sh

Copy this code into this .sh file:

```
#!/bin/bash
#SBATCH -J zcat_wh_18S
#SBATCH -p hour
#SBATCH --error="/home/patrh5/logs/zcat_wh_18S_err.txt"
#SBATCH --output="/home/patrh5/logs/zcat_wh_18S_out.txt"

#This script runs through zcat to create one large .fastq file for all the 18S data for use with Nanoplot.

#18S single fastq barcodes
echo "zcat all 18S files"
zcat /home/patrh5/wh_18S/single_fastq_barcodes/*.fastq.gz | gzip > /home/patrh5/wh_18S/concat_fastqs/concat_raw.fastq.gz
echo "18S file completed"
```

run the script:
`sbatch /home/patrh5/nut_exp/scripts/concat_raw_16S_18S_fastqs.sh

## NanoPlot analysis of my raw project data

Now that I have prepared my concatenated files, I am going to make a script that performs nanoplot analysis of the sequence files, i am going to make separate scripts for each project just to make reproducibility easier.

*Nanoplot of Nutrient Experiment sequencing data*

This script produces Nanoplot output data for the 16S and 18S sequencing data separately, as well as a separate read out for the two negative control PCRS for the 16S and 18S primers.

First, let's make a directory in our project folder for the nanoplot output and then open our new script file with Nano

`mkdir  nut_exp/nanoplot/
`mkdir nut_exp/nanoplot/16S nut_exp/nanoplot/18S nut_exp/nanoplot/16S_NC nut_exp/nanoplot/18S_NC
`nano /home/patrh5/nut_exp/scripts/nanoplot_raw_data.sh

In that script, copy the below text:

```
#!/bin/bash
#SBATCH -J np_nexp_data
#SBATCH -p hour
#SBATCH --error="/home/patrh5/logs/np_nexp_data_err.txt"
#SBATCH --output="/home/patrh5/logs/np_nexp_data_out.txt"

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the nanopack-env and specifiy nanoplot version
conda activate nanopack-env

echo "Plotting 16S data"
NanoPlot --fastq_rich /home/patrh5/nut_exp/concat_fastqs/concat_raw_16S.fastq.gz  --huge --verbose --tsv_stats -o /home/patrh5/nut_exp/nanoplot/16S/

echo "Plotting 18S data"
NanoPlot --fastq_rich /home/patrh5/nut_exp/concat_fastqs/concat_raw_18S.fastq.gz  --huge --verbose --tsv_stats -o /home/patrh5/nut_exp/nanoplot/18S/

echo "Plotting 16S NC file summary"
NanoPlot --fastq_rich /home/patrh5/nut_exp/single_fastq_barcodes/NC_16S.fastq.gz --verbose --tsv_stats -o /home/patrh5/nut_exp/nanoplot/16S_NC/

echo "Plotting 18S NC file summary"
NanoPlot --fastq_rich /home/patrh5/nut_exp/single_fastq_barcodes/NC_18S.fastq.gz --verbose --tsv_stats -o /home/patrh5/nut_exp/nanoplot/18S_NC/

```

Run with:

`sbatch /home/patrh5/nut_exp/scripts/nanoplot_raw_data.sh

*Nanoplot of Ward Hunt Microbiome sequencing data*

This script produces Nanoplot output data for the 18S sequencing data for the Ward Hunt environmental microbiome data. It also produces an output of the 18S PCR negative control.

`mkdir  wh_18S/nanoplot/
`mkdir wh_18S/nanoplot/18S wh_18S/nanoplot/18S_NC
`nano /home/patrh5/wh_18S/scripts/nanoplot_raw_data.sh

In that script, copy the below text:

```
#!/bin/bash
#SBATCH -J np_wh18S_data
#SBATCH -p hour
#SBATCH --error="/home/patrh5/logs/np_wh18S_data_err.txt"
#SBATCH --output="/home/patrh5/logs/np_wh18S_data_out.txt"

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the nanopack-env and specifiy nanoplot version
conda activate nanopack-env

echo "Plotting 18S data"
NanoPlot --fastq_rich /home/patrh5/wh_18S/concat_fastqs/concat_raw.fastq.gz  --huge --verbose --tsv_stats -o /home/patrh5/wh_18S/nanoplot/18S

echo "Plotting 18S NC file summary"
NanoPlot --fastq_rich /home/patrh5/wh_18S/single_fastq_barcodes/NC.fastq.gz --verbose --tsv_stats -o /home/patrh5/wh_18S/nanoplot/18S_NC

```

Run with:

`sbatch /home/patrh5/nut_exp/scripts/nanoplot_raw_data.sh

___

# 7. Filtering ONT data

Once we have got an idea of the quality of our sequencing run, we want to try improve it. We do this first by removing any potential adapter sequences in the data. Generally, it's a good idea to trim adapters (here we are referring to sequencing adapters that bind the DNA to the pores during sequencing and are on the outside of our barcodes) before performing our filtering because adapters are not part of the actual biological sequence and can interfere with downstream analyses like alignment and assembly. After adapter trimming with Porechop, we can then proceed with quality filtering and length filtering of the reads. 

## Adapter and Chimera removal with PoreChop

[PoreChop](https://github.com/bonsai-team/Porechop_ABI/blob/master/README_PORECHOP.md) is a tool for finding and removing known ONT adapters from sequencing reads. 

The benefit of this software is it has a library of known ONT adapter sequences from the Native Barcoding Kit we used, and several other kits. The second benefit is it can help with removal of chimeras (two conjoined reads with an adapter in the middle), by identification of middle adapters and subsequent removal. 

**Important Note: PoreChop has been officially unsupported since 2018**

### 1. Installation

Make sure you are in your 'base' conda environment before creating your new porechop environment:

`mamba create -n porechop-env -c conda-forge -c bioconda -c defaults porechop 
`conda activate porechop-env
`conda list #check that porechop is installed

### 2. How it works

1. Porechop first aligns a subset of reads (default = 10000 change with --check_reads) to all known adapter sets.
2. Adapter sets with at least one high identity match (default = 90%, change with --adapter_threshold) are deemed present in the sample.
3. Identity in this step is measured over the full length of the adapter. E.g. in order to qualify for a 90% match, an adapter could be present at 90% identity over its full length, or it could be present at 100% identity over 90% of its length, but a 90% identity match over 90% of the adapter length would not be sufficient.

**Trim adapters from read ends**
- The first and last bases in each read (default = 150bp, change with --end_size) are aligned to each present adapter set. When a long enough (default = 4, change with --min_trim_size) and strong enough (default = 75%, change with --end_threshold) match is found, the read is trimmed. A few extra bases (default = 2, change with --extra_end_trim), are removed past the adapter to ensure removal.
- Identity in this step is measured over the aligned part of the adapter, not its full length. E.g. if the last 5 bases of an adapter exactly match the first 5 bases of a read, that counts as a 100% identity match and those bases will be trimmed off. This allows Porechop to effectively trim partially present barcodes. 
- The default --end_threshold is low (75%) because false positives (trimming off some sequence that wasn't really an adapter) shouldn't be too much of a problem with long reads, as only a tiny fraction of the read is lost.

**Split reads with internal adapters**
- The entirety of each read is aligned to see if adapters are present in the middle of the reads, indicating a chimera. When a strong enough match is found (default 85%, change with --middle_threshold), the read is split. If the resulting parts are too short (default = less than 1000bp, change with --min_split_read_size), they are discarded.
- The default --middle_threshold (85%) is higher than the default --end_threshold (75%) because false positives in this step (splitting a read that is not chimeric) could be more problematic than false positives in the end trimming step. If false negatives (failing to split a chimera) are worse for you than false positives (splitting a non-chimera), you should reduce this threshold (e.g. --middle_threshold 75).
-Extra bases are also removed next to the hit, and how many depends on the side of the adapter. If we find an adapter that's expected at the start of a read, it's likely that what follows is good sequence but what precedes it may not be. Therefore, a few bases are trimmed after the adapter (default 10, change with --extra_middle_trim_good_side) and more bases are trimmed before the adapter (default 100, change with --extra_middle_trim_bad_side). If the found adapter is one we'd expect at the end of the read, then the "good side" is before the adapter and the "bad side" is after the adapter.
- **It is advisable to discard chimeras from demultiplexed sequencing runs as we can't guarantee the chimera does not contain two sequences from two separate barcodes. We do this with --discard_middle**

**Output**
- If Porechop is run with the output file specified using -o, it will display progress info to stdout. It will try to deduce the format of the output reads using the output filename (can handle .fastq, .fastq.gz, .fasta and .fasta.gz). The --format option can be used to override this automatic detection.
- The --verbosity option will change the amount of progress info:
	- --verbosity 0 gives no progress output.
	- --verbosity 1 (default) gives summary info about end adapter trimming and middle adapter splitting.	
	- --verbosity 2 shows the actual trimmed/split sequences for each read (described more below).
	- --verbosity 3 shows tons of data (mainly for debugging).

### 3. Test run on one file

`porechop --verbosity 1 --discard_middle -i input_reads.fastq.gz -o output_reads.fastq.gz

Let's try this on one of our single fastq barcode files for our nutrient experiment.

First, let's make an output directory for the filtered files

`mkdir /home/patrh5/nut_exp/porechop_output/

Then let's try this command on sample M_C1_16S from the Nutrient Experiment, we are going to specify verbosity 2 here so we can see what reads it produces.

`sbatch porechop --verbosity 2 --discard_middle -i /home/patrh5/nut_exp/single_fastq_barcodes/M_C1_16S.fastq.gz -o /home/patrh5/nut_exp/porechop_output/M_C1_16S.fastq.gz

It seemed a faff to make a script for this one command line, so the output of this will go to a slurm output file, e.g. 'slurm-619736.out'

**Output from M_C1_16S**
Move the output slurm file to my log file:
`mv ~/slurm-619736.out ~/logs/test_porechop_M_C1_16S

Interestingly, PoreChop identified forward and reverse reads for ONT barcode 8, the barcode used for this sample!

` ^[[32mBarcode 8 (reverse)                      100.0      100.0^[[0m`
` ^[[32mBarcode 8 (forward)                      100.0      100.0^[[0m
`#Here ^[[33, and ^[[0m are ANSI escape codes for starting and ending colored text, for some reason the code is showing up rather than colouring the output, possibly to do with slurm?

In total, several barcodes were matched:
```
^[[1m^[[4mTrimming adapters from read ends^[[0m
     SQK-NSK007_Y_Top: ^[[31mAATGTACTTCGTTCAGTTACGTATTGCT^[[0m
  SQK-NSK007_Y_Bottom: ^[[31mGCAATACGTAACTGAACGAAGT^[[0m
             BC05_rev: ^[[31mAAGGTTACACAAACCCTGGACAAG^[[0m
                 BC05: ^[[31mCTTGTCCAGGGTTTGTGTAACCTT^[[0m
             BC08_rev: ^[[31mACGTAACTTGGTTTGTTCCCTGAA^[[0m
                 BC08: ^[[31mTTCAGGGAACAAACCAAGTTACGT^[[0m
             BC11_rev: ^[[31mTCCATTCCCTCCGATAGATGAAAC^[[0m
                 BC11: ^[[31mGTTTCATCTATCGGAGGGAATGGA^[[0m
             BC12_rev: ^[[31mTCCGATTCTGCTTCTTTCTACCTG^[[0m
                 BC12: ^[[31mCAGGTAGAAAGAAGCAGAATCGGA^[[0m
                 BC08: ^[[31mTTCAGGGAACAAACCAAGTTACGT^[[0m
             BC08_rev: ^[[31mACGTAACTTGGTTTGTTCCCTGAA^[[0m
                 BC58: ^[[31mCAGGTTACTCCTCCGTGAGTCTGA^[[0m
             BC58_rev: ^[[31mTCAGACTCACGGAGGAGTAACCTG^[[0m
           NB05_start: ^[[31mAATGTACTTCGTTCAGTTACGTATTGCTAAGGTTAAAAGGTTACACAAACCCTGGACAAGCAGCACCT^[[0m
             NB05_end: ^[[31mAGGTGCTGCTTGTCCAGGGTTTGTGTAACCTTTTAACCTTAGCAATACGTAACTGAACGAAGT^[[0m
           NB08_start: ^[[31mAATGTACTTCGTTCAGTTACGTATTGCTAAGGTTAAACGTAACTTGGTTTGTTCCCTGAACAGCACCT^[[0m
             NB08_end: ^[[31mAGGTGCTGTTCAGGGAACAAACCAAGTTACGTTTAACCTTAGCAATACGTAACTGAACGAAGT^[[0m
           NB11_start: ^[[31mAATGTACTTCGTTCAGTTACGTATTGCTAAGGTTAATCCATTCCCTCCGATAGATGAAACCAGCACCT^[[0m
             NB11_end: ^[[31mAGGTGCTGGTTTCATCTATCGGAGGGAATGGATTAACCTTAGCAATACGTAACTGAACGAAGT^[[0m
           NB12_start: ^[[31mAATGTACTTCGTTCAGTTACGTATTGCTAAGGTTAATCCGATTCTGCTTCTTTCTACCTGCAGCACCT^[[0m
             NB12_end: ^[[31mAGGTGCTGCAGGTAGAAAGAAGCAGAATCGGATTAACCTTAGCAATACGTAACTGAACGAAGT^[[0m
```

To navigate this verbose output it can be useful to use ctrl+W, the search option in nano, if we search reads, and keep re-searching with ctrl+W we can cycle through all the times reads appears in this verbose document and find the amount of reads discarded at the ends of reads:

` 99,430 / 215,961 reads had adapters trimmed from their start (1,286,898 bp removed) #Here 1286898/99430 tells us only 12bp are being removed from each start
` 97,518 / 215,961 reads had adapters trimmed from their end (1,442,636 bp removed) #Here 1442636/97518 tells us only 14bp are being removed from each end

This suggests a short region is being trimmed from each end, most likely leftover barcode reads. As this is unlikely to affect annotation I think it could be worthwile doing this for all reads.

To look at the reads cut from the middle, we can do ctrl+W followed by ctrl+V, which sends us to the end of the nano file where we can find some final results:

It also identified some middle adapters, suggesting chimeras, which is cool!
`1,786 / 215,961 reads were discarded based on middle adapters`
The result was saved to our output file
`Saved result to /home/patrh5/nut_exp/porechop_output/M_C1_16S.fastq.gz`

### 4. Should I do this for all my samples?

Although PoreChop is now discontinued, there has been discussion of [improved seqeunce quality after porechop](https://community.nanoporetech.com/posts/hidden-quality-surprising). The scientific approach would be to run Porechop on all my samples, create a separate output directory and see if this is the case for our reads! So we're going to make a for loop to conduct this analysis for all our reads, as this approach was pretty verbose, we will use `verbose 1` this time...

**Nutrient Experiment Data**
First open your new script file

`nano /home/patrh5/nut_exp/scripts/porechop.sh

Copy in this code into your open nano .sh file:

```
#!/bin/bash
#SBATCH -J porechop_nexp
#SBATCH -p day
#SBATCH --output="/home/patrh5/logs/porechop_nexp_out.txt"

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the porechop-env and specify porechop and python version
conda activate porechop-env
python --version
echo "porechop"
porechop --version

# Define the directory containing the .fastq.gz files
input_dir=/home/patrh5/nut_exp/single_fastq_barcodes

# Define the directory where the filtered reads will be saved
output_dir=/home/patrh5/nut_exp/porechop_output

# Loop over all .fastq.gz files in the input directory
for file in "$input_dir"/*.fastq.gz
do
  # Get the base name of the file (without the directory and extension)
  base_name=$(basename "$file" .fastq.gz)
  
  # Print the base name
  echo "Processing file: $base_name"

  # Define the path to the output file and give them the suffix '_pc' for porechop
  output_file="$output_dir/${base_name}_pc.fastq.gz"

  # Perform the porechop function, no need to unzip the fastq.gz files
   porechop --verbosity 1 --discard_middle -i "$file" -o "$output_file"
done

```

Run our script, note how we set the time frame to 'day' as this may take several hours as there are a lot of files!
`sbatch /home/patrh5/nut_exp/scripts/porechop.sh

**Ward Hunt Environmental Microbiomes**
First open your new script file

`mkdir /home/patrh5/wh_18S/porechop_output
`nano /home/patrh5/wh_18S/scripts/porechop.sh

Copy in this code into your open nano .sh file:

```
#!/bin/bash
#SBATCH -J porechop_wh_18S
#SBATCH -p day
#SBATCH --output="/home/patrh5/logs/porechop_wh_18S_out.txt"

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the porechop-env and specify porechop and python version
conda activate porechop-env
python --version
echo "porechop"
porechop --version

# Define the directory containing the '.fastq.gz' files
input_dir=/home/patrh5/wh_18S/single_fastq_barcodes

# Define the directory where the filtered reads will be saved
output_dir=/home/patrh5/wh_18S/porechop_output

# Loop over all .fastq.gz files in the input directory
for file in "$input_dir"/*.fastq.gz
do
  # Get the base name of the file (without the directory and extension)
  base_name=$(basename "$file" .fastq.gz)
  
  # Print the base name
  echo "Processing file: $base_name"

  # Define the path to the output file and give them the suffix '_pc' for porechop
  output_file="$output_dir/${base_name}_pc.fastq.gz"

  # Perform the porechop function, no need to unzip the fastq.gz files
   porechop --verbosity 1 --discard_middle -i "$file" -o "$output_file"
done

```

Run our script, note how we set the time frame to 'day' as this may take several hours as there are a lot of files!
`sbatch /home/patrh5/wh_18S/scripts/porechop.sh


HERE IS WHERE I GOT TO - 16.02.24
- The plan is to run porechop on all my samples, see the output for how many reads were removed for each sample.
- Then I will need to concatenate the output files for each project and rerun nanoplot to see the impact these changes have had, if any, on sequence quality, it's also cool it's finding chimeras and removing them
- Using this information I can then review if i will use my porechop files
- From there, my options are, put it through porechop-abi to identify any further potential barcode, or continue straight to size filtering, or don't use porechop and go straight to size filtering.


Let's make a new script for each project `concat_porechop.sh`

**Nutrient Experiment**
`nano /home/patrh5/nut_exp/scripts/concat_porechop.sh

Copy in this info:

```
#!/bin/bash
#SBATCH -J zcat_nut_exp_porechop
#SBATCH -p hour
#SBATCH --output="/home/patrh5/logs/zcat_nut_exp_porechop_out.txt"

#This script runs through zcat to create two large .fastq files, containing the 16S and 18S rRNA genes, respectively.

#Concat porechop filtered single fastq 16S barcodes
echo "zcat all 16S files"
zcat /home/patrh5/nut_exp/porechop_output/*16S_pc.fastq.gz | gzip > /home/patrh5/nut_exp/concat_fastqs/concat_porechop_16S.fastq.gz
echo "porechop 16S files completed"

#Concat porechop filtered single fastq 18S barcodes
echo "zcat all 18S files"
zcat /home/patrh5/nut_exp/porechop_output/*18S_pc.fastq.gz | gzip > /home/patrh5/nut_exp/concat_fastqs/concat_porechop_18S.fastq.gz
echo "porechop 18S files completed"
```

Then run the file
`sbatch /home/patrh5/nut_exp/scripts/concat_porechop.sh

**Ward Hunt Microbiomes**

`nano /home/patrh5/wh_18S/scripts/concat_porechop.sh

Copy in this info:

```
#!/bin/bash
#SBATCH -J zcat_wh_18S_porechop
#SBATCH -p hour
#SBATCH --output="/home/patrh5/logs/zcat_wh_18S_porechop_out.txt"

#This script runs through zcat to create a large .fastq file from all the 18S reads

#Concat porechop filtered single fastq 18S barcodes
echo "zcat all 18S files"
zcat /home/patrh5/wh_18S/porechop_output/*fastq.gz | gzip > /home/patrh5/wh_18S/concat_fastqs/concat_porechop.fastq.gz
echo "porechop 18S files completed"
```

Then run the file
`sbatch /home/patrh5/wh_18S/scripts/concat_porechop.sh


### Let's see the impact porechop has on the quality of our data

** Nutrient Experiment **

Make your output directories and script file:
`mkdir nut_exp/nanoplot/16S_pc nut_exp/nanoplot/18S_pc
`nano /home/patrh5/nut_exp/scripts/plotting_porechop.sh

Copy in this text:

```
#!/bin/bash
#SBATCH -J np_nexp_pc_data
#SBATCH -p hour
#SBATCH --output="/home/patrh5/logs/np_nexp_pc_data_out.txt"

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the nanopack-env and specifiy nanoplot version
conda activate nanopack-env

echo "Plotting 16S data"
NanoPlot --fastq_rich /home/patrh5/nut_exp/concat_fastqs/concat_porechop_16S.fastq.gz --huge --verbose --tsv_stats -o /home/patrh5/nut_exp/nanoplot/16S_pc/

echo "Plotting 18S data"
NanoPlot --fastq_rich /home/patrh5/nut_exp/concat_fastqs/concat_porechop_18S.fastq.gz --huge --verbose --tsv_stats -o /home/patrh5/nut_exp/nanoplot/18S_pc/
```

Save, close, and then run the script:
`sbatch /home/patrh5/nut_exp/scripts/plotting_porechop.sh

** Ward Hunt Microbiomes**

Make your output directories and script file:
`mkdir wh_18S/nanoplot/18S_pc
`nano /home/patrh5/wh_18S/scripts/plotting_porechop.sh

Copy in this text:

```
#!/bin/bash
#SBATCH -J np_wh_18S_pc_data
#SBATCH -p hour
#SBATCH --output="/home/patrh5/logs/np_wh_18S_pc_data_out.txt"

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the nanopack-env and specifiy nanoplot version
conda activate nanopack-env

echo "Plotting 18S data"
NanoPlot --fastq_rich /home/patrh5/wh_18S/concat_fastqs/concat_porechop.fastq.gz --huge --verbose --tsv_stats -o /home/patrh5/wh_18S/nanoplot/18S_pc/
```

Save, close, and then run the script:
`sbatch /home/patrh5/nut_exp/scripts/plotting_porechop.sh

** Analysis of PoreChop approach **

- The package appears to have effectively removed some remaining barcode sequences and adapters - need to look into this further
- The quality score improved for several groups by 0.1 and otherwise the read length hasn't been affected suggesting quality has improved.
- Need to check how many reads were removed and how many potential chimeras identified on average for each sample

As such, I am therefore not going to use PoreChop ABI for now as it appears to have formed a large adapter to remove (90bp) which suggests it may not be accurately removing bps based on the consensus approach, I am therefroe going to move porechop-abi to the appendix of this document for now and go straight onto read filtering.

## Filtering by read length and quality score with Chopper

Now we have applied porechop, the next stage is to improve the quality of our dataset by removing small DNA fragments that are unlikely to be our barcodes, long fragments which are unlikely to be amplicons, and low quality reads using[Chopper](https://github.com/wdecoster/chopper) , another package within NanoPack.

Chopper is a rust implementation of NanoFilt + NanoLyse, both originally written in Python. This tool is for filtering and trimming ONT fastq files.

- We want to filter reads <1000 and >3000 to improve specificty of our PCR metabarcoding.
- We also want to filter reads =< Q9 as this seems to be standard protocol in studies (ADD REFS)

### 1. Installation

Let's add this to our nanopack-env conda environment so all of our NanoPack software can be run from the same environment.

`conda activate nanopack-env 
`mamba install -c bioconda chopper #install chopper within our now open nanopack-env
`chopper --version # check for chopper installation

### 2. Chopper Usage

Options

|Argument|Function|
|---|---|
|--headcrop|Trim N nucleotides from the start of a read, default: 0|
|--tailcrop|Trim N nucleotides from the end of a read, default: 0|
|--maxlength|Sets a maximum read length, default: 2147483647|
|-l, --minlength|Sets a minimum read length, default: >1|
|-q, --quality|Sets a minimum Phred average quality score [default: 0]|
|--threads|Number of parallel threads to use [default: 4]|
|--contam|Fasta file with reference to check potential contaminants against|

E.g. `gunzip -c reads.fastq.gz | chopper -q 10 -l 500 | gzip > filtered_reads.fastq.gz

***NOTE: When handling fastq.gz files, we will often have to use pipes to first unzip the file of interest, perform the analysis, and then rezip the final version. This knowledge is sometimes assumed in manuals for bioinformatics software.***
 
We want Chopper to work through all the single fastq barcodes we have created for each project and then move the filtered output into a new separate file. 

**Chopper - Nutrient Experiment**

Make your output directory
`mkdir /home/patrh5/nut_exp/chopper_output

Make the script:
`nano /home/patrh5/nut_exp/scripts/nut_exp_chopper.sh

copy in this text:
```
#!/bin/bash
#SBATCH -J nexp_chopper
#SBATCH -p day
#SBATCH --output=/home/patrh5/logs/nut_exp_chopper_out.txt

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the nanopack-env and specifiy nanoplot version
conda activate nanopack-env

# Define the directory containing the .fastq.gz files
input_dir=home/patrh5/nut_exp/porechop_output

# Define the directory where the filtered reads will be saved
output_dir=home/patrh5/nut_exp/chopper_output

# Loop over all .fastq.gz files in the input directory
for file in "$input_dir"/*.fastq.gz
do
  # Get the base name of the file (without the porechop suffix, directory and extension)
  base_name=$(basename "$file" _pc.fastq.gz)
  
  # Print the base name
  echo "Processing file: $base_name"

  # Define the path to the output file and add a new suffix to signify the reads have been filtered
  output_file="$output_dir/${base_name}_filt.fastq.gz"

  # Filter the reads and save the filtered reads to the output file
  gunzip -c "$file" | chopper -q 9 --minlength 1000 --maxlength 3000 | gzip > "$output_file"
done
```

run script with:
`sbatch /home/patrh5/nut_exp/nut_exp_chopper.sh


**Chopper - Ward Hunt Microbiomes**

Make your output directory
`mkdir /home/patrh5/wh_18S/chopper_output

Make the script:
`nano /home/patrh5/wh_18S/scripts/wh_18S_chopper.sh

Copy in this text:
```
#!/bin/bash
#SBATCH -J wh_18S_chopper
#SBATCH -p day
#SBATCH --output=/home/patrh5/logs/wh_18S_chopper_out.txt

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the nanopack-env and specifiy nanoplot version
conda activate nanopack-env

# Define the directory containing the .fastq.gz files
input_dir=home/patrh5/wh_18S/porechop_output

# Define the directory where the filtered reads will be saved
output_dir=home/patrh5/wh_18S/chopper_output

# Loop over all .fastq.gz files in the input directory
for file in "$input_dir"/*.fastq.gz
do
  # Get the base name of the file (without the porechop suffix, directory and extension)
  base_name=$(basename "$file" _pc.fastq.gz)
  
  # Print the base name
  echo "Processing file: $base_name"

  # Define the path to the output file and add a new suffix to signify the reads have been filtered
  output_file="$output_dir/${base_name}_filt.fastq.gz"

  # Filter the reads and save the filtered reads to the output file
  gunzip -c "$file" | chopper -q 9 --minlength 1000 --maxlength 3000 | gzip > "$output_file"
done
```

run script with:
`sbatch /home/patrh5/wh_18S/wh_18S_chopper.sh

**What is this chopper script doing?**
1. First we set up SBATCH to retrieve the nanopack-env conda environment we created from our home directory install of miniconda
2. The input_dir and output_dir variables are defined to hold the paths to the directory containing the input files and the directory where the output files should be saved, respectively.
3. The `for` loop iterates over all .fastq.gz files in the input directory.
4. Inside the loop, basename "$file" pc.fastq.gz is used to get the base name of the current file (i.e., the file name without the porechop suffix, directory and extension).
5. The output_file variable is defined to hold the path to the output file. The output file name is the base name of the input file with _filt.fastq.gz appended to it.
6. This base_name is printed so I can get a list of how many reads were kept for each sample.
7. The gunzip -c "$file" command decompresses the current file to standard output. The decompressed data is piped to chopper -q 9 -l 1000 --maxlength 2000, which filters the reads. The filtered reads are then piped to gzip > "$output_file", which compresses the data and saves it to the output file.

**A bit more info on the basename command we are using here**

The command base_name=$(basename "$file" .fastq.gz) is used in bash scripting to extract the base name of a file, excluding its directory and extension.

`basename` is a command-line utility that strips directory and suffix from filenames. It takes two arguments: 
1. the first is a pathname
2. the second is a suffix to remove from the file name.

e.g. basename $file .fastq.gz, where:

1. "$file" is a variable that holds the full path of the file. The double quotes are used to handle any spaces or special characters in the file path.
2. .fastq.gz is the suffix that you want to remove from the file name.
___

# 8. Nanoplot analysis of filtering approach

Once we have run these pre-processing steps we will concatenate our samples one more time and look at the impact our filtering has had on our samples.

** Nutrient Experiment **
Let's make a script this time that concats the output files in your `chopper_output directory` and then analyses the concat filtered file and your filtered 16S and 18S NCs all in one script.

Make your output directories for your nanoplot data:
`mkdir /home/patrh5/nut_exp/nanoplot/16S_filt /home/patrh5/nut_exp/nanoplot/18S_filt /home/patrh5/nut_exp/nanoplot/16S_NC_filt /home/patrh5/nut_exp/nanoplot/18S_NC_filt

Make your script
`nano /home/patrh5/nut_exp/scripts/plotting_filt_data.sh

Copy in this text:

```
#!/bin/bash
#SBATCH -J np_nexp_filt_data
#SBATCH -p hour
#SBATCH --output="/home/patrh5/logs/np_nexp_filt_data_out.txt"

#Concat porechop filtered single fastq 16S barcodes
echo "zcat all filtered 16S files"
zcat /home/patrh5/nut_exp/chopper_output/*16S_filt.fastq.gz | gzip > /home/patrh5/nut_exp/concat_fastqs/concat_16S_filt.fastq.gz
echo "filtered 16S files completed"

#Concat porechop filtered single fastq 18S barcodes
echo "zcat all 18S files"
zcat /home/patrh5/nut_exp/chopper_output/*18S_filt.fastq.gz | gzip > /home/patrh5/nut_exp/concat_fastqs/concat_18S_filt.fastq.gz
echo "filtered 18S files completed"

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the nanopack-env and specifiy nanoplot version
conda activate nanopack-env

echo "Plotting filtered 16S data"
NanoPlot --fastq_rich /home/patrh5/nut_exp/concat_fastqs/concat_16S_filt.fastq.gz --huge --verbose --tsv_stats -o /home/patrh5/nut_exp/nanoplot/16S_filt

echo "Plotting filtered 18S data"
NanoPlot --fastq_rich /home/patrh5/nut_exp/concat_fastqs/concat_18S_filt.fastq.gz --huge --verbose --tsv_stats -o /home/patrh5/nut_exp/nanoplot/18S_filt

echo "Plotting filtered 16S NC file summary"
NanoPlot --fastq_rich /home/patrh5/nut_exp/chopper_output/NC_16S_filt.fastq.gz --verbose --tsv_stats -o /home/patrh5/nut_exp/nanoplot/16S_NC_filt

echo "Plotting filtered 18S NC file summary"
NanoPlot --fastq_rich /home/patrh5/nut_exp/chopper_output/NC_18S_filt.fastq.gz --verbose --tsv_stats -o /home/patrh5/nut_exp/nanoplot/18S_NC_filt

```

Save, close, and then run the script:
`sbatch /home/patrh5/nut_exp/scripts/plotting_filt_data.sh

** Ward Hunt Microbiomes**
Let's make a script this time that concats the output files in your `chopper_output directory` and then analyses the concat filtered file and your filtered 18S negative control all in one script.

Make your output directories for your nanoplot data:
`mkdir /home/patrh5/wh_18S/nanoplot/18S_filt /home/patrh5/wh_18S/nanoplot/18S_NC_filt

Make your script
`nano /home/patrh5/wh_18S/scripts/plotting_filt_data.sh

Copy in this text:

```
#!/bin/bash
#SBATCH -J np_wh_18S_filt_data
#SBATCH -p hour
#SBATCH --output="/home/patrh5/logs/np_wh_18s_filt_data_out.txt"

#Concat porechop filtered single fastq 18S barcode samples
echo "zcat all wh_18S files"
zcat /home/patrh5/wh_18S/chopper_output/*.fastq.gz | gzip > /home/patrh5/wh_18S/concat_fastqs/concat_filt.fastq.gz
echo "filtered wh_18S files completed"

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the nanopack-env and specifiy nanoplot version
conda activate nanopack-env

echo "Plotting filtered 18S data"
NanoPlot --fastq_rich /home/patrh5/wh_18S/concat_fastqs/concat_filt.fastq.gz --huge --verbose --tsv_stats -o /home/patrh5/wh_18S/nanoplot/18S_filt

echo "Plotting filtered 18S NC file summary"
NanoPlot --fastq_rich /home/patrh5/wh_18S/chopper_output/NC_filt.fastq.gz --verbose --tsv_stats -o /home/patrh5/wh_18S/nanoplot/18S_NC_filt

```

Save, close, and then run the script:
`sbatch /home/patrh5/wh_18S/scripts/plotting_filt_data.sh

**What do I find out from my NanoPlot data?**

*Nutrient Experiment*

*Ward Hunt Microbiomes*



___


# 9. Taxonomic annotation with Emu
Once we are happy with the filtering we have applied to our sequencing data we can then undertake taxonomic annotation of our reads! This step aims to produce an output abundance table showing the number of reads for a certain taxonomic group in each sample in our study. We will need to do this for the total 16S and 18S samples separately in our nutrient experiment project and for all 18S samples in our wh_18S project.

**References**
[Emu GitLab Page](https://gitlab.com/treangenlab/emu)
[Emu paper](https://www.nature.com/articles/s41592-022-01520-4)
[Description of the Emu EM model](https://www.youtube.com/watch?v=ySt5zIPEb10&t=54s)

**The reasonng behind Emu**
- A read is more likely to be a species that is already in the sample
- Initial estimate of taxonomy, is then re-estimated iteratively as you input more DNA
- To apply this, we apply an expectation-maximization


## The EM model (as I understand it...)

1. Reads are mapped onto our database sequences, we count matches, mismatches, and insertions / deletions to generate an alignment-type probability - how likely is there an error.
2. For each read we then produce an alignment type probability -How likely our read is what is in our taxonomic database
3. Alignment scores are fed into an abundance estimate
4. Which then iteratively cycles back to influence classification likelihood through a positive feedback loop, iteratively. A species is more likely to be the same as other abundant species in the sample. Eventually error will therefore reduce through repeated iterative cycles of classification based on the growing population of assigned reads.
3. This produces a final abundance estimate.


## Using EMU on the NHM HPC

## 1. Installation

### Download the standard database - we need this for 16S

Let's store these files in the group repository (1) so they are available for other group members and (2) as they are large files

1. Install osfclient for install of the Emu database

`cd /mbl/share/workspaces/groups/jungblutlab/Emu

`conda install conda-forge::osfclient # I chose to install this in my base conda this time, in case it is useful for future work

2. Export the database as below:

```
export EMU_DATABASE_DIR=/mbl/share/workspaces/groups/jungblutlab/Emu/standard_db
cd ${EMU_DATABASE_DIR}
osf -p 56uf7 fetch osfstorage/emu-prebuilt/emu.tar
tar -xvf emu.tar

#Note: We specify our EMU_DATABASE_DIR as where we want to install the directory on the HPC. The cd line then ensures we move into that directory before fetching the prebuilt taxonomy from osfstorage
#Note: The tar command extracts our emu.tar zipped file
```

Note: Emu v3.0+ database requirements differ from previous versions. Check you are using the appropriate database for the version you are running. 

Both databases contain identical information: a combination of rrnDB v5.6 and NCBI 16S RefSeq from 17 September, 2020. Taxonomy is also from NCBI on the same date. The resulting database contains 49,301 sequences from 17,555 unique bacterial and archaeal species.

3. Activate appropriate conda environment

We need a conda environment containing python v3.7 and then an installation of Emu

Let's make our conda environment with the correct python v3.7 first

```
conda create -n emu-env python=3.7
conda activate emu-env #activate conda environment
python --version #check that you have installed python
conda install -c conda-forge -c bioconda -c defaults emu
#install the emu package in your emu-env environment
conda -list #check that emu is now installed in your emu-env environment
```

Note: Although not required, we recommend using minimap2 versions >=2.22 to avoid memory leak bug with highly repetitive sequences.

When I installed emu, it was installing minimap2-2.26 

## 2. Alternative Databases

The Emu pre-built databases and files\scripts used to construct the databases are stored on OSF. You can download from the command line using osfclient. If downloading from the command line, first define the EMU_PREBUILT_DB variable.

*Remember to do this in your base conda, not your emu-env as you haven't installed osf here*

for the SILVA v138.1 database we use `export EMU_PREBUILT_DB='silva'`, as below:

```
mkdir /mbl/share/workspaces/groups/jungblutlab/Emu/silva/ # make a working directory for the Silva v138 database
cd /mbl/share/workspaces/groups/jungblutlab/Emu/silva/
osf -p 56uf7 fetch osfstorage/emu-prebuilt/silva.tar
tar -xvf silva.tar
```

### Installing the pre-made PR2 database by Daniel Valout

The first step is to download the PR2 database from the [GitHub repository](https://gitlab.com/treangenlab/emu/-/issues/49)

This should automatically download locally as a tar zip file, we therefore need to send this file to the HPC from your local server.

1. Open PowerShell / cmd on your local PC
2. Type the following scp command, specifying the location of the tar file on your PC:

`scp -J patrh5@orca.nhm.ac.uk C:/local/path/pr2_version_5.0.0_emu_db.tar.gz patrh5@hpc-head-002:/mbl/share/workspaces/groups/jungblutlab/Emu/
3. Fill in your password and authenticator for the orca and for the hpc server (it should be the same password)
4. Back on the NHM HPC, unzip your file:

`tar -xvf /mbl/share/workspaces/groups/jungblutlab/Emu/pr2_version_5.0.0_emu_db.tar.gz
5. This makes a directory called emu_db, to avoid confusion, rename this directory to `pr2_v5_emu_db`

`mv emu_db pr2_v5_emu_db

## 3. Quick review

You should now have installed the standard EMU assignment database, the silva database, and the PR2 database in your desired location, in this case, the group workspace.

## 4. Abundance estimation parameters

| Command | Default | Description |
| --- | --- | --- |
| --type | map-ont | denote sequencer: short-read = 'sr', Pac-Bio = 'map-pb', ONT = 'map-ont' |
| --min-abundance | 0.0001 | generates results with species relative abundance above this value, 0.0001 = 0.01% |
| --db | $EMU_DATABASE_DIR | path to emu database; directory must include the following files: names_df.tsv, nodes_df.tsv, species_taxid.fasta, unique_taxids.tsv |
| --N | 50 | max no of alignments utilized for each read in minimap2 |
| --K | 500M | minibatch size for mapping in minimap2 |
| --output-dir | ./results | directory for output results |
| --output-basename | stem of input_file(s) | basename of all output files saved in your output-dir, default is the basename of your input files | 
| --keep-files | FALSE | keep working files in output-dir |
| --keep-counts | FALSE | include estimated read counts for each species in output |
| --keep-read-assignments | FALSE | output .tsv file with read assignment distributions: each row as an input read; each entry as the likelihood it is derived from the taxa |
| --output-clasified | FALSE | generate a separate output files of unclassified sequences | 
| --threads | 3 | number of threads utilized by minimap2 |

Note: If you are experiencing heavy RAM consumption, first upgrade minimap2 to at least v2.22. If memory is still an issue, try decreasing the number of secondary alignments evaluated for each read (--N).
Note: Estimated read counts are based on likelihood probabilities and therefore may not be integer values.

## 5. Additional commands

### Collapse Taxonomy
The collapse-taxonomy function can be used on any emu output <.tsv> file to generate an additional output collapsed at the desired taxonomic rank. File is output the same folder as the input file, with filename:
 
<input_file>-<rank>.tsv.

Accepted ranks: ['species', 'genus', 'family', 'order', 'class', 'phylum', 'superkingdom']

e.g.:

`emu collapse-taxonomy <file_path> <rank>

### Combine Outputs
The combine-outputs function can be used to create a single table containing all Emu output relative abundances in a single directory. This selects all functions that contain 'rel-abundance' in their filename. Combined table will only include all ranks above specified rank according to this list: *tax_id, species, genus, family, order, class, phylum, superkingdom*

Specified rank must be in this list and in each of the included Emu outputs. Combined table will be created in the provided directory path with the file name <emu-combined-*rank*.tsv> 

`emu combine-outputs <directory_path> <rank>

### Additional parameters:

| Command | Description |
| --- | --- |
| --split-tables | output 2 tables: (1) abundances only at specified rank and (2) taxonomic lineages down to specified rank |
| --counts | output estimated counts rather than relative abundance percentage in combined table. Only includes Emu relative abundance outputs that already have 'estimated counts' |

# 10. Using Emu for my project data

## Pre-requisites
1. As the databases are communal, you need to specify the true directory to the different Emu databases in the 'jungblutlab' shared group workspace, with the `--db` argument
2. Your reads will be in your home directory, so you want to run them using their absolute pathway.
3. You need to be in the emu-env conda environment we created to run the emu command, `conda activate emu-env`
___

# Appendix

## Appendix 1. NanoStat

Before finding NanoPlot, I first tried another software in NanoPack, [NanoStat](https://github.com/wdecoster/nanostat).

[Reference for NanoPack2](https://academic.oup.com/bioinformatics/article/39/5/btad311/7160911?login=false)

Essentially, Nanostat provides the statistics about read quality without the plots. It is a very easy and versatile package, however it is slightly superseded by the ability of NanoPlot to complete these features. 

Basic info on how I used this package:


### 1. Installation

NanoStat requires => Python3

`python --version

Let's create a conda / mamba environment to house the various packages of NanoPack, the suite of ONT analysis software by [Woulter Decoster](https://github.com/wdecoster/nanopack?tab=readme-ov-file)

`mamba create -n nanopack-env -c bioconda nanostat

`conda activate nanopack-env
`conda list

*NanoStat v1.6.0 installed*

### 2. Usage
`Nanostat -h # List of commands

`NanoStat --fastq M_C1_16S.fastq.gz --barcoded --name nut_exp --outdir ~/nut_exp/nanostat/statreports
 
NanoStat --fastq reads.fastq.gz --outdir statreports

### 3. Here's a basic bash script for all the nutrient experiment files in my project

The script below will read every single fastq barcode file for the nutrient experiment, and produce an individual quality report for each file.

`nano ~/nut_exp/nanostat/nut_exp_nanostat.sh

Copy in this text:

```
#!/bin/bash
#SBATCH -p hour


# Loop over all fastq.gz files in the input directory
for file in ~/nut_exp/single_fastq_barcodes/*.fastq.gz
do
  # Get the base name of the file (without the directory and extension)
  base_name=$(basename "$file")

  # Apply NanoStat to the file and save the output to a file
  echo $base_name
  NanoStat --fastq $file --name ~/nut_exp/nanostat/stat_summary/$base_name.tsv --outdir ~/nut_exp/nanostat/stat_summary
  
done

```

## Appendix 2. More info on chimeras

A chimeric read in the context of Oxford Nanopore sequencing is a read that contains two or more distinct sequences (usually from different DNA fragments) that are joined together. This can happen during library preparation when adapters are ligated to DNA fragments. Porechop identifies potential chimeras by looking for adapter sequences in the middle of reads. If it finds an adapter sequence, it splits the read into two separate reads at that point. This effectively removes chimeras by breaking them up into their constituent sequences. However, it's important to note that this method may not catch all chimeras, especially if they were formed without the inclusion of an adapter sequence. More sophisticated methods for chimera detection and removal may be necessary for a thorough analysis. 

[More info:](https://github.com/bonsai-team/Porechop_ABI/blob/master/README_PORECHOP.md#find-matching-adapter-sets)
The entirety of each read is aligned to the present adapter sets to spot cases where an adapter is in the middle of the read, indicating a chimera. When a strong enough match is found (default 85%, change with --middle_threshold), the read is split. If the resulting parts are too short (default less than 1000 bp, change with --min_split_read_size), they are discarded. 

The default --middle_threshold (85%) is higher than the default --end_threshold (75%) because false positives in this step (splitting a read that is not chimeric) could be more problematic than false positives in the end trimming step. If false negatives (failing to split a chimera) are worse for you than false positives (splitting a non-chimera), you should reduce this threshold (e.g. --middle_threshold 75).

Extra bases are also removed next to the hit, and how many depends on the side of the adapter. If we find an adapter that's expected at the start of a read, it's likely that what follows is good sequence but what precedes it may not be. Therefore, a few bases are trimmed after the adapter (default 10, change with --extra_middle_trim_good_side) and more bases are trimmed before the adapter (default 100, change with --extra_middle_trim_bad_side).

## Appendix 3. Uploading and downloading files directly to your PC with SCP and proxyjump

- We can use WinSCP to easily move files between our server and our local PC. The benefit of WinSCP is the GUI means we can drag and drop between servers! In WinSCP we can also rename files manually, which can be a life saver.
- However, WinSCP is (1) glitchy and likes to chuck you out unexpectedly, (2) slow and often times out when transferring large files.
- It would be neater to use SCP directly in the command line. This is slightly complicated by the orca proxy to access the NHM servers. 

Normally, an SCP command looks like:

`scp /path/to/file.txt username@destination_server:~/my_directory

However, as we have the added Orca step we write the SCP command like:

`scp -J 'patrh5@orca.nhm.ac.uk' /path/to/file.txt patrh5@hpc-head-002:~/` - This would send the file to my home directory

To **download** files from the HPC to my desktop we flip the commands:

`scp -J 'patrh5@orca.nhm.ac.uk' patrh5@hpc-head-002:~/my_file.txt .`

Use the '-r' option to transfer a directory

As an example, here is the command I used to scp my zipped ONT run to my personal PC desktop in one command!:

`scp -r -J 'patrh5@orca.nhm.ac.uk' patrh5@hpc-head-002:/mbl/share/workspaces/groups/jungblutlab/Paddy_Nanopore/ONT_1_18S/PH_2023-09-06_ARCTIC_18S.tar.gz .\Desktop\
