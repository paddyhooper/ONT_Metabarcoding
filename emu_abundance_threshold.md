#Emu analysis with different abundance

[Emu paper](https://www.nature.com/articles/s41592-022-01520-4#Sec16)

##Emu algorithm

The Emu algorithm begins by generating alignments between input reads and database sequences with minimap2. 
With these mappings, the following steps are completed: establishment of the initial probabilities, redistribution of the sample composition with an EM algorithm, and then trimming of noise for a final estimation.

The initial probabilities and redistribution of sample composition is complicated but the important section is noise trimming for final estimation. Due to the nature of the probabilistic structure in an EM model, vector F is likely to contain a long tail of low abundance species. To avoid this long list of false positives in the output, any abundance below the set threshold will be modified to 0 at this stage. The default threshold for Emu is an abundance equivalent to 1 read for samples with under 1,000 reads and 10 reads for larger samples. I think this is referring to the optional argument `--min-abundance` which is set at default to 0.0001 which would be 0.01%. However, as PR2 produced no 0.0001 output file in addition to the full results, it suggests that no reads were above this threshold. Therefore, I think I need to decrease this. There are no other variables so i think this is relevant.


##Emu 16S database
The default database of Emu is a combination of rrnDB v5.6 (ref 47) and NCBI 16S RefSeq downloaded on 17 September 2020 (ref 45). Duplicate entries were removed. The resulting database contains 49,301 sequences from 17,555 unique microbial species. 

rrnDB v5.6 was added as it contains species-level taxonomy and a few ambigious bases.

SILVA contains far more sequences, however SILVA does not map completely to the NCBI taxonomy. Greengenes does not have enough species-level annotation and RDP does not perform well in Emu development. Given EMu's reliance on mapping each read to several database sequences, we have found that a smaller, well curated database performs better in EMu. 

Emu is a homology-aware alignment likelihood approach in which read classification probabilities are adaptively updated based on read alignments to multiple references equences and the current community profile estimate. This iterative approach goes beyond simple classification of individual reads and instead uses information gathered from the entire community to enable accurate and robust community profiling despite high error rates in the input sequences. This approach has demonstrated erorr reduction and superior results compared to metagenomic approaches (Curry et al. 2020). 

The main benefits of Emu are:
1. The reduced number of false positives
2. The ability to distinguish genomically similar organisms

While both MetaMaps and Emu are developed for long-read data and incorporate a form of the EM algorithm, their difference is best understood in terms of 'horizontal' alignment in MetaMaps and 'vertical' alignment in Emu. MetaMaps was developed for shotgun metagenomics and can make use of homology information from entire microbial genomes to correctly place individual reads, this enables MetaMaps to skip base-level alignment between reads and reference sequences. Emu, by contrast, is designed for accurate analysis of an individual locus across a very large number of reference records; given that the 16S rRNA gene seuqences may differ between species by only a few bases, Emu operates under conditions in which individual alignment matches and mismatches need to be carefully examined while taking into account the increased error rate of ONT sequencing. Base-level alignment between individual reads and the 16S rRNA gene reference database therefore become necessary, along with an EM approach that uses alignment likelihood model tailored to 16S rRNA gene sequence analysis.

**My concern for 18S datasets:**
Taken directly from the Emu paper discussion:
*"Due to the nature of probabilistic models, Emu creates a long tail of low-abundance species. To avoid this long inaccurate list in the results, the built-in threshold for Emu is the equivalent abundance of 1 read for samples with less than 1,000 reads or 10 reads for anything larger. This means that Emu will not be able to detect microbes with abundance lower than this threshold. This occurs in our synthetic gut mock community, which contains only five Clostridium leptum reads, and thus Emu does not report this species. Therefore, in cases in which the detection of ultra-low-abundance organisms is imperative, Emu would probably not be the best tool."*

Therefore, the optimal abundance cut-off to distinguish between true assingments and noise reduction is an open question for Emu. A future improvement to the model could be to use the statistical information from the sample to establish a minimum abundance threshold of the current somwhat arbitrary cut-off. The fact it is arbritary means I think I can lower it for our protist data, as we may be cutting off valuable data due to the significantly lower read counts in this data.

Another limitation is the reliance on closed reference data. Currently reads that aren't classified are not included in the final output, this is unmined data. 

Conclusion from Emu discussion:
*"The potential for long, single-molecule reads to deliver higher resolution pictures of microbial communities from single-pass sequencing remains enticing, but the high rate of sequencing error is a formidable obstacle. Specifically, although short reads are constrained in sensitivity below the genus level, long reads are not; instead, their difficulty is with specificity. In the case of long reads applied to 16S rRNA amplicon sequencing, Emu represents an important improvement in minimizing this trade-off and it has the potential to show the communities of well-studied environments in a new light. Given that the error profiles are dynamically learned from the input data, Emu has the flexibility to adjust as sequencing technologies develop and would continue to be appropriate for use in novel third-generation sequencing technologies in future applications."*

The only variables I am able to change is the --min-abundance, which generates results with species relative abundnace above this value in addition to full results. 0.0001 is equivalent to 0.01%, if i change this to 0.00001 would it make a difference?
I am adjusting for the difference in read depth between 16S and 18S, samples are going to be less represented due to lower amplification and read depth


Let's rerun the script with this added abundance factor for the Ward Hunt samples for now:

##Script for Ward Hunt environmental microbiomes with 0.00001 abundance parameter

First let's make our working directories for our Emu output
`mkdir /home/patrh5/wh_18S/emu_output/lower_threshold

Then let's make a new log folder for our lower threshold 0.00001 emu outputs

`mkdir /home/patrh5/logs/emu/wh_18S/lower_threshold


Let's make our array script in our nutrient experiment scripts folder
`nano /home/patrh5/wh_18S/scripts/array_emu_18S_lower_threshold.sh

Copy in the script file:

```
#!/bin/bash
#SBATCH -J emu_lt_wh18S
#SBATCH -p day
#SBATCH --output="/home/patrh5/logs/emu/wh_18S/lower_threshold/emu_%A_%a.txt"
#SBATCH --array=1-120

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the Conda environment where Emu is installed
conda activate emu-env

# Define the directory containing the .fastq.gz files
input_dir=/home/patrh5/wh_18S/chopper_output

# Get the name of the input file based on the array index
file=$(ls "$input_dir"/*.fastq.gz | sed -n "${SLURM_ARRAY_TASK_ID}p")

# Get the number of CPU cores
num_cores=$(nproc)

# Run the Emu command with the --threads option
emu abundance "$file" --db /mbl/share/workspaces/groups/jungblutlab/Emu/pr2_v5_emu_db/ --output-dir /home/patrh5/wh_18S/emu_output/lower_threshold/ --min-abundance 0.00001 --keep-counts --keep-read-assignments --threads $num_cores --output-unclassified 

#NOTE: The correct database must be specified, in this case our PR2 database
#NOTE: This script is to test the different reducing the minimum abundance makes on output, if any!
#NOTE: Output unclassified will create a separate output of unclassified reads, showing how many we are losing
```
Run this script
`sbatch /home/patrh5/wh_18S/scripts/array_emu_18S_lower_threshold.sh

Let's look at the output of this run and see how it compares to the previous run. As this is a relatively unintensive task I actually just ran it directly from my home directory! To do this you have to have your conda emu-env activated.

`conda activate emu-env
`mkdir /home/patrh5/wh_18S/emu_output/18S/lower_threshold/rel-abundance
`mkdir /home/patrh5/wh_18S/emu_output/18S/lower_threshold/read-assignment-distributions
`mkdir /home/patrh5/wh_18S/emu_output/18S/lower_threshold/unclassified

`emu combine-outputs /home/patrh5/wh_18S/emu_output/18S/lower_threshold/rel-abundance species

This produces a file in your `rel-abundance` directory called `/home/patrh5/wh_18S/emu_output/18S/lower_threshold/rel-abundance species/emu-combined-species.tsv`

Rename this file to `emu-combined-species-rel-abundance.tsv`
`mv /home/patrh5/wh_18S/emu_output/18S/lower_threshold/rel-abundance species/emu-combined-species.tsv /home/patrh5/wh_18S/emu_output/18S/lower_threshold/rel-abundance species/emu-combined-species-rel-abundance.tsv

#this would include the species and the estimated counts rather than relative abundance



# Script for Nutrient Experiment 18S with 0.00001 abundance threshold 

First let's make our working directories for our Emu output
`mkdir /home/patrh5/nut_exp/emu_output/18S/lower_threshold

Then let's make a new log folder for our lower threshold 0.00001 emu outputs
`mkdir /home/patrh5/logs/emu/18S/lower_threshold/

Let's make our array script in our nutrient experiment scripts folder
`nano /home/patrh5/nut_exp/scripts/array_emu_18S_lower_threshold.sh

Copy in the script file:

```
#!/bin/bash
#SBATCH -J emu_lt_nexp
#SBATCH -p day
#SBATCH --output="/home/patrh5/logs/emu/18S/lower_threshold/emu_%A_%a.txt"
#SBATCH --array=1-41

# Initialize Conda
source /home/patrh5/miniconda3/etc/profile.d/conda.sh

# Activate the Conda environment where Emu is installed
conda activate emu-env

# Define the directory containing the .fastq.gz files
input_dir=/home/patrh5/nut_exp/chopper_output

# Get the name of the input file based on the array index
file=$(ls "$input_dir"/*18S_filt.fastq.gz | sed -n "${SLURM_ARRAY_TASK_ID}p")

# Get the number of CPU cores
num_cores=$(nproc)

# Run the Emu command with the --threads option
emu abundance "$file" --db /mbl/share/workspaces/groups/jungblutlab/Emu/pr2_v5_emu_db/ --output-dir /home/patrh5/nut_exp/emu_output/18S/lower_threshold/ --min-abundance 0.00001 --keep-counts --keep-read-assignments --threads $num_cores --output-unclassified 

#NOTE: The correct database must be specified, in this case our PR2 database
#NOTE: This script is to test the different reducing the minimum abundance makes on output, if any!
#NOTE: Output unclassified will create a separate output of unclassified reads, showing how many we are losing
```
Run this script
`sbatch /home/patrh5/nut_exp/scripts/array_emu_18S_lower_threshold.sh

Will need to remove the duplicate sample as before:

`rm /home/patrh5/nut_exp/emu_output/18S/lower_threshold/M_TZ1_18S_filt.fastq_*
`mv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/M_TZ1_b18S_filt.fastq_read-assignment-distributions.tsv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/M_TZ1_18S_filt.fastq_read-assignment-distributions.tsv
`mv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/M_TZ1_b18S_filt.fastq_rel-abundance.tsv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/M_TZ1_18S_filt.fastq_rel-abundance.tsv
`mv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/M_TZ1_b18S_filt.fastq_unclassified.tsv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/M_TZ1_18S_filt.fastq_unclassified.tsv

I then need to move all the files into seperate subdirectories:
`mkdir /home/patrh5/nut_exp/emu_output/18S/lower_threshold/rel-abundance /home/patrh5/nut_exp/emu_output/18S/lower_threshold/read-assignment-distributions /home/patrh5/nut_exp/emu_output/18S/lower_threshold/unclassified  

`mv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/*abundance.tsv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/rel-abundance/
`mv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/*distributions.tsv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/read-assignment-distributions/
`mv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/*.fa /home/patrh5/nut_exp/emu_output/18S/lower_threshold/unclassified

I now want to make two merged output tables for all my samples, at the sample level:
`conda activate emu-env
`emu combine-outputs /home/patrh5/nut_exp/emu_output/18S/lower_threshold/rel-abundance/ species
`mv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/rel-abundance/emu-combined-species.tsv /home/patrh5/nut_exp/emu_output/18S/lower_threshold/rel-abundance/emu-combined-species_rel_abun_0_00001.tsv 

Let's also make my table at the previous abundance level for comparison:
`emu combine-outputs /home/patrh5/nut_exp/emu_output/18S/rel-abundance/ species
`mv /home/patrh5/nut_exp/emu_output/18S/rel-abundance/emu-combined-species.tsv /home/patrh5/nut_exp/emu_output/18S/rel-abundance/emu-combined-species_rel_abun_0_0001.tsv 

We now have two files to compare the output of the different abundance levels and see if they are any different. I am going to export them using winscp to my local PC and compare them in Excel as this is a quick way to check the data.

#Comparing output for nutrient experiment
So, this makes no difference to the taxonomic annotation.  I think this is what they were talking about when they say: "in a larger sample (1,000 reads or more), a taxon needs to be represented by at least 10 reads to be included". This threshold helps to reduce noise in the abundance estimation and focus on the taxa that are most likely to be truly present in the sample.

I do think however, it's cool to have the unclassified results output as well. so i think i should re-run my 16S analysis with the unclassified files.

In the file titled "rel-abundance-threshold-x.tsv", the value that is replacing "x" means any species with an estimated abundance below that value is marked as not present in the sample and its abundance proportion is redistributed in the sample. Depending on the number of reads in your sample and if you have adjusted the default min-abundance-threshold parameter, Emu may return 2 files with differing minimum abundance thresholds. - https://gitlab.com/treangenlab/emu/-/issues/45 

Could i include unclassified reads in my final abundance count?
https://gitlab.com/treangenlab/emu/-/issues/33 
Correct, the unassigned reads are not included abundance estimation, but it is included in the "estimated counts" columns (when using --keep-counts). If you prefer to calculate your abundance including unassigned reads, you can calculate the proportions from the estimated counts.



- Could i run unclassified runs through something?
- If i had reads that were present in a certain threshold of samples, could i then pull these out and then run them?

Estimated counts is generated by simply multiplying the estimated abundance by the total number of mapped reads.  - https://gitlab.com/treangenlab/emu/-/issues/42 

Need to make a new version of my emu analysis with 16S as `--output-unclassified` needed

I am going to make version folders for my output files
v1 = initial test of emu, only produced relative abundance files
v2 = re-ran all my different project sample directories adding in the read-assingment-distributions.tsv files for each sample and also keep-counts to keep the value for estimated counts
lower_threshold = this was testing out changing the -abundance-threshold to a lower amount, i also added in adding the -unclassified reads as a fasta file for this section, which i think is useful. Therefore, I am going to re-run all analysis in my original script with this argument added. However, I do not think it is necessary to add the abundance-threshold argument as I don't think it changes anything.

Therefore, I am going to edit my original ONT_native_barcoding_analysis.md file to add in this label - i have rerun all these scripts on Friday 23rd Feb

Output of these files will be my output.

#Things to consider in results and discussion

Difference in read depth between 16S samples and 18S samples in read counts across all 40 samples
Abundance level I chose to use for 18S - do i use this for 16S?
number of unclassified reads produced in our sample set
Do I want to do anything with the unclassified reads


Results: nutrient experiment

- Do i want to rarefy to a certain sample depth for comparisons of diversity? 
- Difference between sample groups in alpha diversity - increased diversity with addition of nutrients?
- Difference in taxonomic composition ?
- Taxonomic diversity
- Chlorophyll A changes? 

Results: ward hunt microbiomes

- Difference in the different environments
- Water, soil, crust, mats
- Individual environments
- Phylogenetic analysis of certain groups?
- Analyis of unclassified material?

Novelty - first published use of Emu for 18S protist data?
- first ont paper from Ward Hunt Island


Need to compare output to see if it has actually changed anything between the two abundance levels, maybe check like 10 smaples
then try get data into my phyloseq object on R for wh_18S project?
