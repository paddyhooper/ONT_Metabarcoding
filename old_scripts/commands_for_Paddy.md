# ONT metagenomic pipeline draft

This file is written in [Markdown](https://www.markdownguide.org/basic-syntax/) and includes various commands written in Bash and [AWK](https://en.wikipedia.org/wiki/AWK). Not all commands have been tested and should only be used as template/reference. Environments are created using Conda.

Hope this helps - any questions, please feel free to drop me an email.

Silvia

**Combine all FASTQ files per sample:**  

```bash
mkdir Single_fastq_barcodes/
bash combine_fastqs.sh
# each line of the script above should look like:
# cat YOUR_ONT_RUNFOLDER/fastq_pass/barcode01/*.fastq.gz >> Single_fastq_barcodes/YOUR_SAMPLE_NAME_HERE_FOR_BARCODE01.fastq.gz
```
 
**Filter out short reads:**  

```bash
mkdir Logs/
mkdir Length_filtered_ONT_data/
conda activate nanofilt_2.8.0
for f in `ls Single_fastq_barcodes/*.fastq.gz`; do gunzip -c $f | NanoFilt -l 200 | gzip > Length_filtered_ONT_data/$(basename $f); done
conda deactivate
```
 
**Trim ONT adapters:**  

```bash
mkdir Adapter_trimmed_ONT_data/
conda activate porechop
for f in `ls Length_filtered_ONT_data/*.fastq.gz`; do porechop -i $f -t 4 --min_split_read_size 200 -o Adapter_trimmed_ONT_data/$(basename $f .gz) > Logs/porechop_$(basename $f .fastq.gz).log; done
# the line below is needed to fix the format of porechop stdout log
for f in `ls Logs/porechop*`; do esc=$'\e' ; sed "s/$esc\[[0-9;]*m//g" $f | grep -v -P $'\r' | sponge $f; done
```
 
**Mask low-complexity regions:**  
This complicated one-liner is needed to pass a FASTA to dustmasker (which does not like FASTQs) and then convert the output multi-line FASTA to a single-line FASTA.
 
```bash
mkdir LCR_masked_ONT_data/
for f in `ls Adapter_trimmed_ONT_data/*.fastq`; do ~/ncbi-blast-2.13.0+/bin/dustmasker -in <(awk 'NR%4==1||NR%4==2' $f | tr '@' '>') -hard_masking -outfmt fasta | awk '/^>/ { if(NR>1) print ""; printf("%s\n",$0); next; } { printf("%s",$0);} END {printf("\n");}' > LCR_masked_ONT_data/$(basename $f .adapter_trimmed.fastq).masked.fasta ; done

```
 
**Perform taxonomic classification:**  
Note to self: the reason for using options `--minimum-hit-groups` and `--confidence` in Kraken2 lies in [this thread](https://github.com/DerrickWood/kraken2/issues/403). To use slurm, please refer to the NHM guide on Teams and on [the official documentation](https://slurm.schedmd.com/documentation.html).

```bash
mkdir Kraken2_results/
ls -1 LCR_masked_ONT_data/* > Scripts/hpc_fastq_filelist.config
sbatch Scripts/hpc_kraken_nt.sh Scripts/hpc_fastq_filelist.config Kraken2_results/
```

NCBI Kraken DB: `/mbl/share/workspaces/groups/database/nt-2022-11-14_kraken/ncbi_nt/`
We don't currently have a Kraken2-database at NHM for PR2 and Silva, you would have to create them yourself - please find the instructions [here](https://github.com/DerrickWood/kraken2/wiki/Manual).
 
Kraken script for the HPC cluster:

``` 
#!/bin/bash
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=s.salatino@nhm.ac.uk
#SBATCH --job-name=kraken_nt
#SBATCH --output=Logs/hpc_kraken_%A_%a.log
#SBATCH --no-requeue
#SBATCH --partition=gpu
#SBATCH --cpus-per-task=10
#SBATCH --mem-per-cpu=64g
#SBATCH --array=1-12%2
 
# get input parameters
config_file=$1     #(file containing tab-separated fields)
 
# define variables
fasta_path=$(awk "NR==$SLURM_ARRAY_TASK_ID" $config_file | cut -f 1)
out_folder=$(dirname $fasta_path)
sample_name=$(basename $fasta_path .fastq.masked.fasta)
 
# run the commands
source /home/silvs/miniconda3/bin/activate kraken2
cmd="kraken2 --confidence 0.05 --minimum-hit-groups 3 --threads 10 --db /mbl/share/workspaces/groups/database/nt-2022-11-14_kraken/ncbi_nt/ --report ${out_folder}/${sample_name}_kraken_report.tab --output ${out_folder}/${sample_name}_kraken_out.tab $fasta_path"
echo $cmd
$cmd
echo "Processing sample $sample_name done. Status:" $?
```
 
**Generate the Krona plot:**  

```bash
mkdir Krona_results/
conda activate krona
for f in `ls Kraken2_results/*_out.tab`; do ktImportTaxonomy -q 2 -t 3 $f -o Krona_results/$(basename $f).html; done
conda deactivate
```

