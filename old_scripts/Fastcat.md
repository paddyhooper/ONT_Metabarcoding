Fastcat

A set of simple utilities for creating summaries from Oxford Nanopore data

# 1. Installation

`mamba create -n fastcat-env -c conda-forge -c bioconda -c nanoporetech fastcat

`conda activate fastcat-env

`conda list #check fastcast is now installed

# 2. Arguments

|Argument|Function|Default|
|---|---|---|
|-a|minimum read length to output (excluded reads remain listed in summaries)||
|-b|maximum read length to pout (excluded reads remain listed in summaries)||
|-d|demultiplex, separate barcoded samples using fastq geader information. Option value is top-level output directory||
|-f|file summary, per-file summary output directory?||
|-H|--reheader,re-write fastq header as SAM tags||
|-q|minimum q score to output (exluded reads remain listed in summaries)||
|-r|per-read summary output||
|-s|sample name - adds a sample name column||
|-x|search directories recursively for .fastq files||


-x can be used for recursive writing from the ONT barcode folders


# 3. Output files

1. The program writes the input sequences to stdout in .fastq format to be recommpressed with gzip
2. The per-read.txt is a tab-seperated file with columns of reads
3. The per-file.txt is a tab separated file of the different filenames
4. length-hist - read length histogram
5. quality-hist - read mean base-quality score histogram

When data is demultiplexed one such file will be written to the demultiplexed samples' directories. 

#4. Trying FastCat for my files

Make a directory in my nut_exp folder `fastcat

`mkdir /~/nut_exp/fastcat

Copy across some of my 16S barcode folders so I can play around with them

`cp -r ~/nut_exp/16S/ont2/barcode01 ~/nut_exp/fastcat/barcode01

This is a 16S rRNA gene metabarcoding run containing the sample M_N4_16S

Let's try run fastcat on this sample and see what happens

`fastcat -a 1000 -b 2000 -q 9 -s M_N4_16S ~/nut_exp/fastcat/barcode01

A lot happened when  I pressed this... it opened the whole file

It didn't produce any output

Let's try without the commands


fastcat doesn't seem to be working, maybe I need to unzip

`for f in '~/nut_exp/fastcat/barcode01/*.fastq.gz'; do gunzip -c $f | fastcat -a 1000 -b 2000 -q 9 -s M_N4_16S | gzip > ~/nut_exp/fastcat/filt_ont_data/$(basename $f); done

`gunzip -c reads.fastq.gz | chopper -q 10 -l 500 | gzip >> filtered_reads.fastq.gz


mkdir Logs/
mkdir Length_filtered_ONT_data/


conda activate nanofilt_2.8.0
for f in `ls Single_fastq_barcodes/*.fastq.gz`; do gunzip -c $f | NanoFilt -l 200 | gzip > Length_filtered_ONT_data/$(basename $f); done
conda deactivate


#GitHub script for doing my work

#!/bin/bash

# Define the directory containing the fastq files
input_dir="~/nut_exp/fastcat"

# Define the directory to store the processed files
output_dir="~/nut_exp/fastcat/filtered"

# Create the output directory if it doesn't exist
mkdir -p $output_dir

# Loop over all fastq files in the input directory
for file in $input_dir*.fastq.gz
do
  # Get the base name of the file (without directories and extension)
  base_name=$(basename $file .fastq.gz)

  # Unzip the file, pipe it into fastcat, and zip the output
  gunzip -c $file | fastcat -a 1000 -b 2000 -q 9 -s M_N4_16S | gzip > $output_dir/${base_name}_filtered.fastq.gz
done


i don't like fastcat, let's try another package

