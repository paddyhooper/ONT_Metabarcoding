# Using EMU to analyse 16S and 18S rRNA gene reads

**References**
[GitHub Page](https://gitlab.com/treangenlab/emu)
[EMU paper](https://www.nature.com/articles/s41592-022-01520-4)
[Description of the EMU model](https://www.youtube.com/watch?v=ySt5zIPEb10&t=54s)


**The premise**
- A read is more likely to be a species that is already in the sample
- Initial estimate of taxonomy, is then re-estimated iteratively as you input more DNA
- To apply this, we apply an expectation-maximization


# The EM model (as I understand it...)

1. Reads are mapped onto our database sequences, we count matches, mismatches, and insertions / deletions to generate an alignment-type probability - how likely is there an error.
2. For each read we then produce an alignment type probability -How likely our read is what is in our taxonomic database
3. Alignment scores are fed into an abundance estimate
4. Which then iteratively cycles back to influence classification likelihood through a positive feedback loop, iteratively. A species is more likely to be the same as other abundant species in the sample. Eventually error will therefore reduce through repeated iterative cycles of classification based on the growing population of assigned reads.
3. This produces a final abundance estimate.


# Using EMU on the NHM HPC

# 1. Installation

## Download the standard database - we need this for 16S

Let's store these files in the group repository so they are available and as they are probably large files

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

# 2. Alternative Databases

Our pre-built databases and files\scripts used to construct the databases are stored on OSF. You can download from our OSF UI or from the command line using osfclient. If downloading from the command line, first define the EMU_PREBUILT_DB variable.

*Remember to do this in your base conda, not your emu-env as you haven't installed osf here*

for the SILVA v138.1 database we use `export EMU_PREBUILT_DB='silva'`, as below:

```
mkdir /mbl/share/workspaces/groups/jungblutlab/Emu/silva/ # make a working directory for the Silva v138 database
cd /mbl/share/workspaces/groups/jungblutlab/Emu/silva/
osf -p 56uf7 fetch osfstorage/emu-prebuilt/silva.tar
tar -xvf silva.tar
```

# Installing the pre-made PR2 database by Daniel Valout

The first step is to download the PR2 database from the [GitHub repository](https://gitlab.com/treangenlab/emu/-/issues/49)

This should automatically download locally as a tar zip file, we therefore need to send this file to the HPC from your local server.

1. Open PowerShell / cmd on your PC
2. Type the following scp command, specifying the location of the tar file on your PC:

`scp -J patrh5@orca.nhm.ac.uk C:/local/path/pr2_version_5.0.0_emu_db.tar.gz patrh5@hpc-head-002:/mbl/share/workspaces/groups/jungblutlab/Emu/
3. Fill in your password and authenticator
4. Unzip tar file:

`tar -xvf /mbl/share/workspaces/groups/jungblutlab/Emu/pr2_version_5.0.0_emu_db.tar.gz
5. this makes a directory called emu_db, to avoid confusion, rename this directory

`mv emu_db pr2_v5_emu_db

# 3. Quick review

You should now have installed the standard EMU assignment database, the silva database, and the PR2 database in your desired location, in this case, the group workspace.


# 4. Abundance estimation parameters

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

# 5. Additional commands

## Collapse Taxonomy
The collapse-taxonomy function can be used on any emu output <.tsv> file to generate an additional output collapsed at the desired taxonomic rank. File is output the same folder as the input file, with filename:
 
<input_file>-<rank>.tsv.

Accepted ranks: ['species', 'genus', 'family', 'order', 'class', 'phylum', 'superkingdom']

e.g.:

`emu collapse-taxonomy <file_path> <rank>

## Combine Outputs
The combine-outputs function can be used to create a single table containing all Emu output relative abundances in a single directory. This selects all functions that contain 'rel-abundance' in their filename. Combined table will only include all ranks above specified rank according to this list: *tax_id, species, genus, family, order, class, phylum, superkingdom*

Specified rank must be in this list and in each of the included Emu outputs. Combined table will be created in the provided directory path with the file name <emu-combined-*rank*.tsv> 

`emu combine-outputs <directory_path> <rank>

## Additional parameters:

| Command | Description |
| --- | --- |
| --split-tables | output 2 tables: (1) abundances only at specified rank and (2) taxonomic lineages down to specified rank |
| --counts | output estimated counts rather than relative abundance percentage in combined table. Only includes Emu relative abundance outputs that already have 'estimated counts' |

_________________________

# Using EMU for my data

Calculate relatuve abundance for ONT single-end 16S reads:

`emu abundance example/full_length.fa

This will create a relative abundance .tsv file in a ./ results folder.

Output names are:
full_length_rel-abundance.tsv

# Essentials

1. As the databases are communal, you need to specify the true directory to the different Emu databases in the 'jungblutlab' shared group workspace, with the `--db` argument

2. Your reads will be in your home directory, so you want to run them using their absolute pathway.

3. You need to be in the emu-env conda environment we created to run the emu command, `conda activate emu-env`

#What else do we need to do first to our sequence data? 
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

To start with I am going to try this approach on two of my 16S samples.

I need a script that:

1. Concatenates the barcode and applies an 'alias' sample name using a sample_sheet.csv
2. Filters reads to remove anything 1000 < x < 2000  
3. Removes adapters
4. Uses the Kraken2 database for taxonomic analysis 


trim barcodes was on, on the Minknow software, do i still need to trim adapters?

#FastQC

#PoreChop

