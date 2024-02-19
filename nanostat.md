# NanoStat

## Background
[GitHub](https://github.com/wdecoster/nanostat)

[Reference for NanoPack2](https://academic.oup.com/bioinformatics/article/39/5/btad311/7160911?login=false)


We use NanoStat and not CRAMINO as we have .fastq files

## 1. Installation

NanoStat requires => Python3

`python --version

Let's create a conda / mamba environment to house the various packages of NanoPack, the suite of ONT analysis software by [Woulter Decoster](https://github.com/wdecoster/nanopack?tab=readme-ov-file)

`mamba create -n nanopack-env -c bioconda nanostat

`conda activate nanopack-env
`conda list

*NanoStat v1.6.0 installed*

## 2. Usage
`Nanostat -h # List of commands

`NanoStat --fastq M_C1_16S.fastq.gz --barcoded --name nut_exp --outdir ~/nut_exp/nanostat/statreports
 
NanoStat --fastq reads.fastq.gz --outdir statreports

## 3. Here's a basic bash script for all the nutrient experiment files

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

## 4. Here's a basic bash script for all the ward hunt 18S files

The script below will read every single fastq barcode file for the nutrient experiment, and produce an individual quality report for each file.

`nano ~/wh_18S/nanostat/wh_18S_nanostat.sh

Copy in this text:

```
#!/bin/bash
#SBATCH -p hour


# Loop over all fastq.gz files in the input directory
for file in ~/wh_18S/single_fastq_barcodes/*.fastq.gz
do
  # Get the base name of the file (without the directory and extension)
  base_name=$(basename "$file")

  # Apply NanoStat to the file and save the output to a file
  echo $base_name
  NanoStat --fastq $file --name ~/wh_18S/nanostat/stat_summary/$base_name.tsv --outdir ~/nut_exp/nanostat/stat_summary
  
done

```

Do i want to make an average for 16S and 18S reads by concatenating all 16S and 18S reads into one file? ??

___

# Chopper

## Background

[Chopper GitHub page](https://github.com/wdecoster/chopper)

A rust implementation of NanoFilt + NanoLyse, both originally written in Python. This tool is for filtering and trimming ONT fastq files.

With this step we are aiming to improve the quality of our sample reads

We want to filter reads <1000 and >2000 to improve specificty of our PCR metabarcoding.

We also want to filter reads < Q9 - this is standard protocol

## 1. Installation

Let's try add this to our NanoPack-env conda environment, it would be nice to have all of the software from this suite in the same environment

`conda activate nanopack-env 
`mamba install -c bioconda chopper
`conda list # check for chopper installation

*chopper v0.7.0 installed*

## 2. Usage

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

Example for one fastq:

`gunzip -c reads.fastq.gz | chopper -q 10 -l 500 | gzip > filtered_reads.fastq.gz


## 3. Filtering my nutrient experiment data

Let's first test out the programme on a copy of one of my single_fastq_barcodes:

`cp ~/nut_exp/single_fastq_barcodes/M_C1_16S.fastq.gz ~/nut_exp/

`gunzip -c ~/nut_exp/M_C1_16S.fastq.gz | chopper -q 10 -l 1000 --maxlength 2000 | gzip > M_C1_16S_filt.fastq.gz


We can test how well this worked using the NanoStat package above. The output printed below shows our filtering has worked as all reads are above Q10 quality score.

`NanoStat --fastq M_C1_16S_filt.fastq.gz

```
*#OUTPUT*
General summary:
Mean read length:                1,445.0
Mean read quality:                  15.7
Median read length:              1,450.0
Median read quality:                17.6
Number of reads:               152,836.0
Read length N50:                 1,451.0
STDEV read length:                  76.6
Total bases:               220,845,108.0
Number, percentage and megabases of reads above quality cutoffs
>Q5:    152836 (100.0%) 220.8Mb
>Q7:    152836 (100.0%) 220.8Mb
>Q10:   152836 (100.0%) 220.8Mb
>Q12:   143063 (93.6%) 206.7Mb
>Q15:   117471 (76.9%) 169.7Mb
Top 5 highest mean basecall quality scores and their read lengths
1:      32.7 (1426)
2:      32.6 (1440)
3:      32.4 (1467)
4:      32.0 (1453)
5:      31.9 (1447)
Top 5 longest reads and their mean basecall quality score
1:      2000 (18.0)
2:      2000 (10.8)
3:      2000 (10.6)
4:      1998 (17.3)
5:      1998 (13.2)
```

## 4. Scaling up our approach to a whole directory

As before, it would be much quicker if we can now input an entire folder of our .fastq.gz files and then move the filtered files into a new directory.

To do this we need another 'for' loop. To keep stuff tidy, let's first make a new directory called filtered_reads

`mkdir ~/nut_exp/filtered_reads

Here's a script for my nutrient experiment files, let's make a new script in our nutrient experiment folder called nut_exp_chopper.sh and then copy the text below in:

`nano ~/nut_exp/nut_exp_chopper.sh

```
#!/bin/bash
#SBATCH -p hour

# Define the directory containing the .fastq.gz files
input_dir=~/nut_exp/single_fastq_barcodes

# Define the directory where the filtered reads will be saved
output_dir=~/nut_exp/filtered_reads

# Loop over all .fastq.gz files in the input directory
for file in "$input_dir"/*.fastq.gz
do
  # Get the base name of the file (without the directory and extension)
  base_name=$(basename "$file" .fastq.gz)

  # Define the path to the output file
  output_file="$output_dir/${base_name}_filtered.fastq.gz"

  # Filter the reads and save the filtered reads to the output file
  gunzip -c "$file" | chopper -q 9 -l 1000 --maxlength 2000 | gzip > "$output_file"
done
```

Here is how many reads chopper kept for my nutrient experiment files, the files were read in the order they appear in the directory:
  GNU nano 4.8                                                                                                  slurm-605806.out
Kept 152836 reads out of 215961 reads
Kept 1838 reads out of 9473 reads
Kept 114818 reads out of 159277 reads
Kept 2628 reads out of 8729 reads
Kept 127989 reads out of 167385 reads
Kept 4880 reads out of 12566 reads
Kept 93079 reads out of 121210 reads
Kept 921 reads out of 6464 reads
Kept 117399 reads out of 149073 reads
Kept 849 reads out of 5948 reads
Kept 115309 reads out of 154494 reads
Kept 1123 reads out of 8735 reads
Kept 107499 reads out of 154733 reads
Kept 1219 reads out of 8326 reads
Kept 81700 reads out of 117584 reads
Kept 768 reads out of 5774 reads
Kept 126954 reads out of 153319 reads
Kept 359 reads out of 2112 reads
Kept 67882 reads out of 92217 reads
Kept 647 reads out of 5671 reads
Kept 99158 reads out of 133846 reads
Kept 854 reads out of 6555 reads
Kept 95887 reads out of 117515 reads
Kept 955 reads out of 7444 reads
Kept 130197 reads out of 164287 reads
Kept 624 reads out of 6809 reads
Kept 155684 reads out of 196327 reads
Kept 328 reads out of 4660 reads
Kept 94977 reads out of 124588 reads
Kept 1047 reads out of 7032 reads
Kept 138753 reads out of 185810 reads
Kept 466 reads out of 5239 reads
Kept 121719 reads out of 149595 reads
Kept 637 reads out of 3525 reads
Kept 3113 reads out of 6262 reads
Kept 117186 reads out of 153679 reads
Kept 7984 reads out of 12344 reads
Kept 94018 reads out of 132377 reads
Kept 7880 reads out of 11379 reads
Kept 110898 reads out of 148572 reads
Kept 4749 reads out of 6993 reads
Kept 150 reads out of 2001 reads
Kept 46 reads out of 2888 reads
Kept 101782 reads out of 126674 reads
Kept 29235 reads out of 38868 reads
Kept 100247 reads out of 122715 reads
Kept 44885 reads out of 54588 reads
Kept 106651 reads out of 127789 reads
Kept 78661 reads out of 93948 reads
Kept 118887 reads out of 138630 reads
Kept 41968 reads out of 51928 reads
Kept 112069 reads out of 133641 reads
Kept 19074 reads out of 25685 reads
Kept 102866 reads out of 124686 reads
Kept 69916 reads out of 83506 reads
Kept 102326 reads out of 127435 reads
Kept 53527 reads out of 65259 reads
Kept 121530 reads out of 142464 reads
Kept 56739 reads out of 70279 reads


## 5. Repeating the script for my wh_18S files

This time I want to add a line that echos the name of each file to the slurm output, this way I know how many reads were taken from each file

*ADD SCRIPT HERE*

### A bit of info on the basename command

The command base_name=$(basename "$file" .fastq.gz) is used in bash scripting to extract the base name of a file, excluding its directory and extension.

`basename` is a command-line utility that strips directory and suffix from filenames. It takes two arguments: 
1. the first is a pathname
2. the second is a suffix to remove from the file name.

e.g. basename $file .fastq.gz, where:

"$file" is a variable that holds the full path of the file. The double quotes are used to handle any spaces or special characters in the file path.

.fastq.gz is the suffix that you want to remove from the file name.

# Porechop_ABI

[GitHub page](https://github.com/bonsai-team/Porechop_ABI/tree/master)

[Reference](https://www.biorxiv.org/content/10.1101/2022.07.07.499093v1)

## Background

Porechop_ABI is an extension of Porechop (which has been unsupported since 2018), the purpose is to process adapter sequences in ONT reads.

The difference with the initial version of Porechop is that Porechop_ABI does not use external knowledge or databases for the adapters. Adapters are discovered directly from the reads using approximate k-mers counting and assembly. These sequences can then be used for trimming using standard Porechop options.

The software is able to report a combination of distinct sequences if a mix of adapters is used. It can also be used to check whether a dataset has already been trimmed out or not, or to find leftover adapters in datasets that have been previously processed with Guppy.

## Why are we using it?

We shouldn't have any barcodes left in our samples as these are demultiplexed and removed as part of the MinKNOW software pipeline. This saves us a lot of time. However, it could be useful to check if we have any adapters leftover in our data.

The software gives us the option to scan our reads for Adapters without removing them from our reads. This is a useful first step.

```
porechop_abi -abi --guess_adapter_only -i input_reads.fastq.gz -o output_reads.fastq porechop_abi -abi -go -i input_reads.fastq -v 0 -o output_reads.fastq.gz
```

## Warnings from Porechop_ABI

1. Low frequency warning: This warning means that the k-mers used to build the adapter sequences do not show a clear over-representation. In this case, the result of the algorithm is questionable, because the signal is not reliable. This may happen when the input reads are already trimmed, for example.
2. Poor consensus warning: This warning is activated when Porechop_ABI finds more than two distinct start (or end) adapters and when each of them is found in less than 30% reads.

## 1. Installation

Make sure to first return to your base conda level if you are still in an active conda environment

`conda deactivate

Install porechop_abi in a new conda environment

`mamba create -n porechop_abi-env -c conda-forge -c bioconda -c defaults porechop_abi
`conda activate porechop_abi-env

*porechop_abi v0.5.0 installed* 

## 2. Adapter inference only
This command-line, with the additional -go flag, allows to only guess and print the adapter sequences from the reads. It then stops the execution of the program, without trimming the reads.

`porechop_abi -abi --guess_adapter_only -i input_reads.fastq.gz -o output_reads.fastq

Let's try it on one read 

`porechop_abi -abi --guess_adapter_only -i ~/nut_exp/M_C1_16S_filt.fastq.gz -o ~/nut_exp/M_C1_16S_filt_pc.fastq.gz 


I got this output:

/!\      The adapters currently found are not all identical.
/!\      A consensus will be made.
/!\      Launching 20 additional runs to build a consensus.

Rebuild adapters:

Start
Consensus_1_start_(100.0%)
TAGAGTTTGATCATGGCTCAGGATGAACGCTGGCGGTATGCTTAACACATGCAAGTCGAACGGAACTCTTCGGAGTTTAGTGGCGGACGGGTGAGTAACGCG

End
Consensus_1_end_(100.0%)
GCGTTACTCACCCGTCCGCCACTAAACTCCGAAGAGTTCCGTTCGACTTGCATGTGTTAAGCATACCGCCAGCGTTCATCCTGAGCCATGATCAAACTCTA

## 3. Removal of adapter sequences
This command line with the --ab_initio or --abi flag allows to first guess the adapters, add the adapters to the list and then run Porechop as usual. 

Let's give it a go on that same file

`porechop_abi -abi -i ~/nut_exp/M_C1_16S_filt.fastq.gz -o ~/nut_exp/M_C1_16S_filt_pc.fastq.gz 


This did remove adapters, it also identified adapters in the middle of reads, suggesting chimeras

## 4. Info on Chimeras
https://github.com/bonsai-team/Porechop_ABI/blob/master/README_PORECHOP.md#find-matching-adapter-sets

Split reads with internal adapters
The entirety of each read is aligned to the present adapter sets to spot cases where an adapter is in the middle of the read, indicating a chimera. When a strong enough match is found (default 85%, change with --middle_threshold), the read is split. If the resulting parts are too short (default less than 1000 bp, change with --min_split_read_size), they are discarded.

The default --middle_threshold (85%) is higher than the default --end_threshold (75%) because false positives in this step (splitting a read that is not chimeric) could be more problematic than false positives in the end trimming step. If false negatives (failing to split a chimera) are worse for you than false positives (splitting a non-chimera), you should reduce this threshold (e.g. --middle_threshold 75).

Extra bases are also removed next to the hit, and how many depends on the side of the adapter. If we find an adapter that's expected at the start of a read, it's likely that what follows is good sequence but what precedes it may not be. Therefore, a few bases are trimmed after the adapter (default 10, change with --extra_middle_trim_good_side) and more bases are trimmed before the adapter (default 100, change with --extra_middle_trim_bad_side). If the found adapter is one we'd expect at the end of the read, then the "good side" is before the adapter and the "bad side" is after the adapter.

Here is a real example of the "good" and "bad" sides of an adapter. The adapter is in the middle of this snippet (SQK-NSK007_Y_Top at about 90% identity). The bases to the left are the "bad" side and their repetitive nature is clear. The bases to the right are the "good" side and represent real biological sequence.


Yes, Porechop, a tool for adapter trimming for Oxford Nanopore reads, does have the ability to remove chimeric reads.

A chimeric read in the context of Oxford Nanopore sequencing is a read that contains two or more distinct sequences (usually from different DNA fragments) that are joined together. This can happen during library preparation when adapters are ligated to DNA fragments.

Porechop identifies potential chimeras by looking for adapter sequences in the middle of reads. If it finds an adapter sequence, it splits the read into two separate reads at that point. This effectively removes chimeras by breaking them up into their constituent sequences.

However, it's important to note that this method may not catch all chimeras, especially if they were formed without the inclusion of an adapter sequence. More sophisticated methods for chimera detection and removal may be necessary for a thorough analysis. 


# Re-checking our quality score

We want to see if our trimming and adapter removal has improved the quality of our dataset, after all this is what we set out out to do. 


1. I can check quality with NanoStat or i could potentially use the NanoPlot which would give me a better overview
2. NanoPlot
3. NanoComp - https://github.com/wdecoster/nanocomp 

## Let's try NanoPlot while we're here

Let's install NanoPlot in our NanoPack environment, to keep things tidy

`conda activate nanopack-env
`conda install -c bioconda nanoplot
`conda list

NanoPlot creates:
a statistical summary
a number of plots
a html summary file

This script performs data extraction from Oxford Nanopore sequencing data in the following formats:

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

## NanoPlot Usage

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

#Let's try run NanoPlot on a fasta file

`NanoPlot --fastq ~/nut_exp/M_C1_16S.fastq.gz

This works it makes lots of outputs and includes interesting information about our run thanks to the rich fastq file which contains info on the run. 

This works, but it would be a huge amount of information to produce these plots for every sample, perhaps it would be more interesting to make a concatenated file for different variations of our samples:

1. Concatenated file for all samples in ONT1: ont1_concat.fastq.gz
2. Concatenated file for all samples in ONT2: ont2_concat.fastq.gz
3. Concatenated file for all samples in WH_18S chapter: wh_18S_concat.fastq.gz
4. Concatenated file for all 16S samples in nut_exp chapter: nut_exp_16S_concat.fastq.gz
5. Concatenated file for all 18S samples in nut_exp chapter: nut_exp_18S_concat.fastq.gz

Let's make a script for to make all these files. We will not include the unclassified data in this run. 

*Concatenating zipped files*

- The cat command you've written is attempting to concatenate all gzipped FASTQ files from each barcode subdirectory under ~/ont1_fastq/fastq_pass/ into a single gzipped FASTQ file ~/total_ont1.fastq.gz.
- However, using cat to concatenate gzipped files will not work as expected because gzip files have a specific format with a header and footer around the compressed data.
- If you concatenate gzip files using cat, you'll end up with multiple headers and footers in the resulting file, which will cause problems when you try to decompress or read the file.
- Instead, you should use zcat or gunzip -c to decompress the data to standard output and then pipe that output to gzip to compress it again into a single file:

`nano concat_ONT_runs.sh

```
#!/bin/bash
#SBATCH -J combine_fastqs_nut_exp_ont1
#SBATCH -p hour

zcat ~/ont1_fastq/fastq_pass/barcode*/*.fastq.gz | gzip > ~/ont1_concat.fastq.gz
echo "ont1_concat.fastq.gz created"

zcat ~/ont2_fastq/fastq_pass/barcode*/*.fastq.gz | gzip > ~/ont2_concat.fastq.gz
echo "ont2_concat.fastq.gz created"

echo "Creating nanoplot output"
```

2. Make a script for the nanoplot for the concatenated sequence runs




`nano nanoplot_ONT_runs.sh

```
#!/bin/bash
#SBATCH -J nanoplot
#SBATCH -p hour

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the nanopack-env and specifiy nanoplot version
conda activate nanopack-env

NanoPlot --fastq_rich ont1_concat.fastq.gz --huge --verbose --tsv_stats -o ~/on>
echo "ONT 1 report finished"
NanoPlot --fastq_rich ont2_concat.fastq.gz --huge --verbose --tsv_stats -o ~/on>
echo "ONT 2 report finished"

```

#If this works, it might suggest we need to use zcat for our single fastq barcodes we created for each sample? 

Do i need to run conda in my script commands

I think i need to recreate my single fastq barcode files with zcat

Let's next week restart from the beginning with a big new script 

BTW you made your concatenated ont1 and ont2 files


# Emu taxonomic annotation

https://gitlab.com/treangenlab/emu 