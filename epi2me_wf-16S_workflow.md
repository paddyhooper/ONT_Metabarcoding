# EPI2ME Labs wf-16S

This Markdown document was created by Patrick M. Hooper on 13/02/2024.

# Preamble

Firstly, the material in this document is my adaptation of the ONT epi2me lab's [wf-16S](https://github.com/epi2me-labs/wf-16s/tree/master?tab=readme-ov-file#FAQs) workflow for taxonomic classification of single reads from amplicon-targeted sequencing for my purposes at the NHM. This workflow allows you to taxonomically classify 16S rRNA and 18S rRNA amplicons using default or custom databases. The default databases include: NCBI targeted loci: 16S rDNA, 18S rDNA, ITS #however this 18S data looks specified for fungi, [here](https://www.ncbi.nlm.nih.gov/refseq/targetedloci/). The output includes an abundance table with counts per taxa for all samples, interactive Sankey and sunburst plots to explore taxonomic lineages, and an abundance barplot housed in an HTML report

Secondly, this markdown document assumes you have copied your ONT sequencing run from the NHM GridION onto your group directory on the HPC. For the jungblut group this is /mbl/share/workspaces/groups/jungblutlab/

***Very Important Sidenote ***
*I copied my sequencing data into the jungblutlab from the GridION using scp into my folder 'Paddy_Nanopore'. Within this, you can then find the two sequencing runs, ONT_1_18S (which contains the 18S data for the environmental microbiomes chapter (except soil, deep mats and WHIS)) and ONT_2 (which contains the nutrient experiment data, soil, deep mat biofilms, and WHIS1).*

***These folders should never be modified!!***

Thirdly, this markdown document mainly uses SBATCH scripts designed for submitting jobs to the NHM HPC servers. If you are unfamiliar with the NHM HPC, please read Robert Foster's HPC user guide and or my markdown file: [NHM_HPC_slurm.md](ADD LINK HERE)

Fourthly, you will want to use ['screen'](https://linuxize.com/post/how-to-use-linux-screen/) for some of your bigger processing steps, this gives you the possibility of making a separate temporary terminal which you can then 'attach' and 'detach' from, meaning you can send a command, detach, and close your terminal, all while the command still runs in your screen. When you think it's done, you just re-attach to the screen to continue working. 
___

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

Taxonomic assignment can be through:

1. minimap2 (alignment based) = default option
2. kraken2 (k-mer based) = faster processing

### Minimap2

Minimap2 provides better resolution, but can take significantly more time. 

`nextflow run epi2me-labs/wf-16s --fastq test_data/case01 --classifier minimap2`

### Kraken2

Kraken2 provides the fastest method for taxonomic classification. Supports real-time analysis.

Then, Bracken is used to provide an estimate of the species / genus abundance in the sample.

*K-mers refers to all the possible substrings of length k that are possible in a string of nucleotides, e.g. a dinucleotide is a kmer where k=2. When we talk about all kmers to talk about all the possible sequences of that length. So for example, when K=2 all the possible kmers are: AA AT AC AG TA TT TC TG CA CT CC CG GA GT GC GG. K is usually bigger than 2, so we can talk about all 4mers (256 of them), all 6mers (4096 of them), all 7mers (16,384 of them) etc.*


## Build a custom database for analysis

How can I use custom databases? - If you want to run the workflow using your own Kraken2 database, you'll need to provide the database and an associated taxonomy dump. For a custom Minimap2 reference database, you'll need to provide a reference FASTA (or MMI) and an associated ref2taxid file. For a guide on how to build and use custom databases, take a look at our article on how to run wf-16s offline:

https://labs.epi2me.io/how-to-meta-offline/

How to select the correct databases:
https://labs.epi2me.io/metagenomic-databases/


## 4. Prepare output

The main output of the wf-16S pipeline is the `wf-16s-report.html`. Which can be found in the output directory. It contains a summary of read statistics, the taxonomic composition of the sample and some diversity metrics. The results shown in the report also can be customized, with the report option arguments listed above.

The workflow output also contains Kraken and bracken reports for each sample. 

Additionally, the ‘species-abundance.tsv’ is a table with the counts of the different taxa per sample. 

You can use the flag `include_kraken2_assignments` to include a per sample TSV file that indicates how each input sequence was classified as well as the taxon that has been assigned to each read. This TSV file will only be output on completion of the workflow.

___

# Prerequisites

1. Nextflow - Pipeline management software
2. Docker or Singularity - This is complicated at the NHM as Docker does not work with HPCs very easily, but there is a way round this for NHM users below.

# 1. Installation

## Install Nextflow

This workflow uses nextflow to manage compute and software resources. Therefore let's first install [nextflow](https://www.nextflow.io/)

```
cd ~
java -version

#Output:
#openjdk version "11.0.13" 2021-10-19
#OpenJDK Runtime Environment JBR-11.0.13.7-1751.21-jcef (build 11.0.13+7-b1751.21)
#OpenJDK 64-Bit Server VM JBR-11.0.13.7-1751.21-jcef (build 11.0.13+7-b1751.21, mixed mode)

#create a nextflow directory in the home directory
curl -s https://get.nextflow.io | bash
ls #you should now see nextflow in your directory

#test your nextflow is installed
 ./nextflow run hello
```

## Install the wf-16S pipeline

Now we have nextflow installed, we can use it to install the wf-16S nextflow pipeline directly from epi2me-labs

`./nextflow run epi2me-labs/wf-16s --help 

A demo dataset is provided for testing of the workflow. It can be downloaded using: 

`wget https://ont-exd-int-s3-euwst1-epi2me-labs.s3.amazonaws.com/wf-16s/wf-16s-demo.tar.gz
`tar -xzvf wf-16s-demo.tar.gz

We have downloaded a zipped file wf-16S-demo.tar.gz and then unzipped it creating a new directory 'wf-16S-demo', in this directory we have a file test_data containing three barcodes and an unclassified folder.

## Using nextflow on the NHM HPC

Normally, you would now install a singularity or docker to carry out package management during your nextflow pipeline. This is complex at the NHM and therefore an easier get around is to instead use the pre-built NHM '.config' files created by Thomas Creedy in the NHM Bioinformatics Team. Here's a link to his [guide](https://github.com/NHM-London-Bioinformatics/pipelines-sop0).

**Thomas' rules for using nextflow on the NHM servers:**

- Always run a pipeline for a specific set of data in its own working directory. Nextflow does most of its workings in a specific `work` directory that you can specify, it also creates a `.nextflow` and `.nextflow` log files in the current working directory, and these may clash with other nextflow directories running in the same directory. 
- Always run a pipeline using [screen](https://linuxize.com/post/how-to-use-linux-screen/). Nextflow dispatches slurm jobs while the main program continues running and reporting progress, but it can take hours so being able to close the screen is invaluable. This is a good tip for all big data analysis!
- Always specify the version `-r` when running a nextflow pipeline. When you want to compare a new run to a previous dataset, use the same version. Otherwise, we generally recommend running the most recent and often most stable version.
- Always keep your own record of the exact command you run, so you can replicate the analysis with future data.
- All pipelines can take as input a sample sheet in TSV form. These can be created in Excel or Google Sheets. This is their recommendation for inputting sample information.
- Always use absolute file paths so your script can be run from any directory.

**Queue selection and configuration**

When we run a nextflow command, two arguments we usually need to provide are (1) the location of our 'config' file and (2) our 'profile' type. For our work, we are going to make our own copy of Thomas' kindly prepared NHM config file in our home directory (to get round access rights). This NHM.config file optimizes the nextflow pathway for the NHM HPC. If  you are interested in understanding this further please see the Appendix to this document. For now, all we need to really know is if we specify this config file in our Nextflow command, we can run our desired workflow.

**Making our personal config file**
To get round access rights, we are going to make a copy of Thomas' NHM.config file in our personal home directory. ***Make sure to not edit or remove any files in the Nextflow group workspace.***

`cp /mbl/share/workspaces/groups/nextflow/NHM.config ~
`mv NHM.config PH_NHM.config #here I am just renaming the file so I know it's my copy
`mkdir ~/nextflow_cache/ # this makes a directory we can store our cache information in on our home directory

Open the copy of the config file in your directory with nano.

`nano PH_NHM.config

Change the following lines from the previous directory to your desired personal directory.

Edit these lines...

```
singularity.cacheDir = '/mbl/share/workspaces/groups/nextflow/cache/singularity'
singularity.libraryDir = '/mbl/share/groups/nextflow/cache/singularity>
conda.cacheDir = '/mbl/share/workspaces/users/patrh5/cache/conda'
```

To...

```
singularity.cacheDir = '/mbl/share/workspaces/users/patrh5/nextflow_cache/singularity'
singularity.libraryDir = '/mbl/share/workspaces/users/patrh5/nextflow_cache/singularity'
conda.cacheDir = '/mbl/share/workspaces/users/patrh5/nextflow_cache/conda'
```

This will hopefully reroute the singularity library and cache to my home directory which should be writeable by my nextflow pipelines. The rest of the .config file tells it to use slurm and send the files to the CPU nodes. 

Now we have made our personal config file, we will specify it in our nextflow commands using the `-c` argument and specify the absolute directory of our personal config file, e.g. `-c ~/PH_NHM.config`

We also need to specify the queue we want to join on the NHM server, e.g. hour, day, week, etc. Most nextflows won't need more than the day queue:

`-profile genericcpuhour, singularity \ #note here how Thomas' approach requires we specify singularity here. If you were using docker you would specify that here instead.

## Running the wf-16S nextflow pipeline
Now we have sorted out all our pre-requisites, we can test the pipeline on the test tutorial data we downloaded earlier. This test data is three barcode directories containing the raw loose .fastq files (not concatenated). All nexflow commands use the `nextflow run <pipeline_name>` command. The full command to run this looks like:

```
~/nextflow run epi2me-labs/wf-16s \
--fastq ~/wf-16s-demo/test_data/ \
-c /mbl/share/workspaces/users/patrh5/PH_NHM.config \
-profile genericcpuhour,singularity

#-fastq specifies which files to run the pipeline on
#-c specifies where to find our config file
#-profile this specifies which profile inside our config file to use, genericcpuhour tells it to use the HPC-CPU-node for 1 hour, and tells nextflow to use singularity, not docker.
```

## Nextflow Arguments

Adapting the wf-16S nextflow pipeline for 16S analysis requires use of further arguments in the Nextflow command.

## Input options

Nextflow parameter name | Type | Description | Help | Default |
|--- | --- | --- | --- | --- |
|fastq|string|FASTQ files to analyze|This accepts one of three cases (i) path to the single FASTQ file, (ii) the path to a top-level directory containing FASTQ files, (iii) path to a directory containing one level of sub-directories which contain FASTQ files, in the first and second case, a sample name can be supplied with --sample, in the third case, the data is assumed to be multiplexed with the names of the sub-directories a barcodes. In this case a sample sheet can be provioded with --sample sheet | |
|classifier|strig|Kraken" or Minimap2 workflow|Use Kraken2 for fast classification and minimap2 for finer resolution|minimap2|
|analyse_unclassified|boolean|Analyse unclassified reads from input directory. By default the workflow will not process reads in the unclassified directory.|If selected and the input is a multiplex directory the workflow will also process the unclassified directory|False|
|exclude host|string|A FASTA or MMI file of the host reference. Reads that align will be excluded from the analysis| | |

## Sample Options

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

## Running wf-16S for my ONT data
To run nextflow with my samples I needed to create [sample sheets](LINK TO SAMPLE SHEETS) to specify my sample IDs.

I need a script that:
1. Concatenates the barcode and applies an 'alias' sample name using a [sample_sheet.csv](ADD LINK TO MY SAMPLE SHEETS]
2. Filters reads to remove anything 1000 < x < 2000  

Let's try using the default taxonomic assigner first, minimap for my 16S samples for the nutrient experiment:

`screen -S 16S_minimap #always open a screen for running iong commands

For analysis our own data, we will use the HPC job scheduler. To do this we need to write a script containing our desired commands and then 'submit' this job using the `sbatch` command, as below.

First, create a blank .sh document to write your script in, in the command line:

`nano wf-16S_minimap_16S.sh

Then, copy the below code and right hand click on your open nano document to copy the commands across, save and exit the nano document once you're finished. The '#' commands at the top are essential for job scheduling scripts.

```
##!/bin/bash
#SBATCH -J wf-16S_minimap_16S
#SBATCH -p day

~/nextflow run epi2me-labs/wf-16s \
--fastq ~/nut_exp/16S/ont2/ \
-c /mbl/share/workspaces/users/patrh5/PH_NHM.config \
-profile genericcpuday,singularity \
--sample_sheet ~/nut_exp/16S_sample_sheet.csv \
--database_set ncbi_16s_18s \
--min_len 1000 \
--max_len 2000 \
--min_read_qual 9 

# -c specifies my config file, as discussed above
# -profile specifies using a singularity and tells nextflow which job scheduler to use.
# --sample_sheet specifies the location of my metadata
# --database_set tells nextflow to use the premade ncbi_16S_18S database
# --min_len = remove sequences below 1000 bp
# --max_len = remove sequences above 2000 bp
# --min_read_qual = remove any nucleotides below a quality score of 9 (more info in appendix)
```

To run this command, we use:

`sbatch wf-16S-minimap_16S.sh

If this is running correctly, you will get a message saying SLURM job no. submitted

## Running Nextflow wf-16S with Kraken2

Next, i tried analysing my 16S data with Kraken2 and the Silva taxonomic database:

`nano wf-16S-kraken_16S.sh

Copy below commands into nano file:

```
##!/bin/bash
#SBATCH -J wf-16S_kraken_16S
#SBATCH -p day

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

Save and close the nano file and run the script:
`sbatch wf-16S-kraken_16.sh

# Conclusions

The wf-16S pipeline is a quick and effective way to produce taxonomic data from 16S metabarcoding data produced with ONT sequencing. The data visualization is very useful for a first review of the sequencing output providing information on sequencing quality and taxonomic assignments. 

The main limitation for my work was the limited analysis for protists with the available 18S databases. The manual provides information on how to create custom databases within Kraken2 but I found this to be a complicated process for data outside of adapting the available Kraken databases or NCBI databases. I was hoping to use the PR2 database for my 18S analysis but was advised this would be quite complex due to the differences in database structure. There are definitely alternative approaches but i decided to try [EMU](ADD REF TO MY EMU SCRIPT) for my taxonomic analysis as a prebuilt PR2 database was available for this software. 
___

# Appendix - NHM.config file

This config sets the path to the NHM [singularity cache](https://github.com/NHM-London-Bioinformatics/pipelines-sop/blob/main/singularity_cache.md) directory and specifies that all processes should be executed by slurm. It then sets four profiles, corresponding to the four different NHM queues, all using CPUs rather than GPUs. As the NHM HPC slurm configuration is set to dispatch to CPU by default, as then no specific CPU dispatch instructions are included. 

**Queue specification**
The config file specifies a map (aka dict or hash) that comprises 3 unique queue options, such that successive attempts will be dispatched to queues with longer running times. Namely, attempts 1 and 2 are dispatched to the hour queue, 3 and 4 the day queue and 5 the week queue.

**Task specification**
The config file overwrites the default starting cpus, memory and time for different processes (tasks) - processes are classified by labels denoting broad levels of computational usage. All processes in eager are marked as belonging to one of these labels, and inherit that label's resource configuration.

**NHM HPC singularity cache**
Nf-core pipelines can source the software for each process from either conda, singularity or docker. We strongly recommend using singularity, as this is installed on the NHM HPC and is more robust than conda. To avoid nextflow downloading singularity images for every different process for every different user, we've set up a central singularity cache that can be read by all users. Using the NHM.config file (see the configs page) automatically points nextflow to this cache location. Note that not everyone can write to the cache: if in the rare instance you run a pipeline that includes processes and singularity images not already cached, you may get an error. In this case, please contact us.

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