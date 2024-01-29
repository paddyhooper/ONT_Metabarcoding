# Making a Kraken 2 database for PR2

https://github.com/DerrickWood/kraken2/blob/master/docs/MANUAL.markdown#custom-databases 

# Step 1: Download and Install Kraken 2 in an environment called kraken2-env

`mamba create -n kraken2-env -y -c conda-forge -c bioconda -c defaults kraken2

`conda activate kraken2-env

I forgot I also need bracken so I have installed this like so:

`conda install bioconda::bracken 

Kraken 2 databases are large, I might make my kraken directory in the group workspace as it is too large to make a database in my home directory


# Kraken 2 requirements

Disk Space: Construction of a Kraken 2 database requires approximately 100 GB of disk space.

Memory: To run efficiently, Kraken 2 requires enough free memory to hold the database (primarily the hash table) in RAM. While this can be accomplished with a ramdisk, Kraken 2 will by default load the database into process-local RAM; the --memory-mapping switch to Kraken 2 will avoid doing so. The default database size is 29 GB (as of Jan. 2018), and you will need slightly more than that in RAM if you want to build the default database.

Nextwork connectivity: Kraken 2' standard database build and download commands expect unfettered FTP and rsync access to the NCBI FTP server. 

Dependencies: By default, Kraken 2 will attempt to use the dustmasker or segmasker programs provided as part of NCBI's BLAST suite to mask low complexity regions


# Kraken 2 Databases

A Kraken 2 database is a a directory containing at least 3 files:

- hash.k2d: contains the minimizer to taxon mappings
- opts.k2d: contains information about the options used to build the database
- taxo.k2d: contains taxonomy information used to build the database

None of  these 3 files are in a human-readable format. Other files may also be present as part of the database build process, but can be removed after successful build of a database

When we eventually want to interact with Kraken 2, you should not have to directly reference any of these files, but rather simply provide the name of the directory in which they are stored. 

Kraken 2 allows the use of **custom databases** as well their standard databases.

# Standard Kraken 2 database

To create the standard Kraken 2 database, you can use the following command:

```
kraken2-build --standard --db $DBNAME

#Repalce $DBNAME with your preferred database name/location. Please note that the database will use approximately 100 GB of disk space during creation, therefore specify the group storage rather than your home directory

```

This will download the NCBI taxonomic information, as well as the complete genomes in RefSeq for the bacterial , archael and viral domains, along with human genome and UniVec_Core. After downloading all this data, the build process begins, this can be the most time-consuming step. 

You can multithread this with the addition of 

` kraken2-build --standard --threads 24 --db $DBNAME

# Custom Databases

To build a custom database 

1. Install a taxonomy:

Usually, you will just use the NCBI taxonomy, which can be easily downloaded using:

`kraken2-build --download-taxonomy --db $DBNAME

This downloads the accession number to taxon maps, as well as the taxonommic name and tree information from NCBI. These files can be found in $DBname/taxonomy/. If you need to edit the taxonomy, edits can be made with names.dmp and nodes.dmp files in this directory, you may also need to modify the *.accession2taxid* files appropriately.

Some of the standard sets of genomic libraries have taxonomic information associated with them, and don't need the accession number to taxon maps to build the database successfully. 

These libraries include all those available through the --download-library option. If you are not using custom sequences, you may want to skip downloading of the accession number to taxon maps, by specifying 

` --skip-maps` to the `kraken2-build --download-taxonomy` command

2. Install a reference library:

##Standard genome reference libraries
Several sets of standard genomes / proteins are made easily avaiable through kraken2-build

protozoa: RefSeq complete protozoan genomes/proteins
fungi: RefSeq complete fungal genomes/proteins
bacteria: RefSeq complete bacterial genomes/proteins
archaea: RefSeq complete archaeal genomes/proteins

To download and install any of these, use the --download-library, e.g.

kraken2-build --download-library protozoa --db $protozoa-db

Equally, multiple libraries can be downloaded into a database prior to building by issuing multiple kraken2-build commands, e.g.:

kraken2-build --download-library archaea --db $DBNAME
kraken2-build --download-library viral --db $DBNAME

However, this has not actually bult the database. We still need to use the --build command, this just downloads the libraries we want.

##Adding specific genomes
Sequences must be be added in a FASTA file
Each sequence's ID must contain the NCBI accession number to allow Kraken 2 to lookup the correct taxa, or an explicit assignment of the taxonomy ID using kraken2:taxid

Sequences not downloaded from NCBI may need their taxonomy information assigned explicitly. This can be done using the string kraken:taxid|XXX in the sequence ID, with XXX replaced by the desired taxon ID. For example, to put a known adapter sequence in taxon 32630 ("synthetic construct"), you could use the following:

```
>sequence16|kraken:taxid|32630  Adapter sequence
CAAGCAGAAGACGGCATACGAGATCTTCGAGTGACTGGAGTTCCTTGGCACCCGAGAATTCCA
```

The kraken:taxid string must begin the sequence ID or be immediately preceded by a pipe character (|). Explicit assignment of taxonomy IDs in this manner will override the accession number mapping provided by NCBI.

We then add the specific genome like:

`kraken2-build --add-to-library chr1.fa --db $DBNAME

Note that if you have a list of files to add, you can do something like this in bash:
```
for file in chr*.fa
do
    kraken2-build --add-to-library $file --db $DBNAME
done
```


Once your **library** is finialised, you need to build the database, this can be done with the command:

kraken2-build --build --db $DBNAME

By default, the values of k-mer and minimizer length are 35 and 31, respectively (or 15 and 12 for protein databases). These values can be explicitly set with the --kmer-len and --minimizer-len options, however. Note that the minimizer length must be no more than 31 for nucleotide databases, and 15 for protein databases. Additionally, the minimizer length must be no more than the k-mer length. There is no upper bound on the value of k, but sequences less than k bp in length cannot be classified.

##Special Databases - 16S databases

To support some common use cases, we provide the ability to build Kraken 2 databases using data from various external databases. These external databases may not follow the NCBI taxonomy, and so we've provided mechanisms to automatically create a taxonomy that will work with Kraken 2 (although such taxonomies may not be identical to NCBI's).

To build one of these "special" Kraken 2 databases, use the following command:

`kraken2-build --db $DBNAME --special TYPE #where the TYPE string is one of the database names listed below.

At present, the "special" Kraken 2 database support we provide is limited to pre-packaged solutions for some public 16S sequence databases, but this may grow in the future.

16S databases:

Greengenes (Kraken 2 database name: greengenes), using all available 16S data.
RDP (Kraken 2 database name: rdp), using the bacterial and archaeal 16S data.
SILVA (Kraken 2 database name: silva), using the Small subunit NR99 sequence set.

##Masking low-complexity regions

Low-complexity sequences, e.g. repeated codon regions are known to occur in many different organisms and are typically less informative for use in alignments. The BLAST program often masks these low quality sequences by default. Using this masking can help prevent false positives in Kraken 2 results and so we have added this functionality as a default option in Kraken 2.

Kraken 2 uses two programs to perform low-complexity sequence masking, both available from NCBI: dustmasker, for nucleotide sequences, and segmasker, for amino acid sequences. These programs are available as part of the NCBI BLAST+ suite. If these programs are not installed on the local system and in the user's PATH when trying to use kraken2-build, the database build will fail. Users who do not wish to install these programs can use the --no-masking option to kraken2-build in conjunction with any of the --download-library, --add-to-library, or --standard options; use of the --no-masking option will skip masking of low-complexity sequences during the build of the Kraken 2 database.



# How does this translate to ONT pipeline?

ONT workflow: https://labs.epi2me.io/how-to-meta-offline/ 

The structure and composition of the databases

![wf-16S](https://labs.epi2me.io/static/3b86d43d1d03ca55455ad5ab4e0be948/09ede/databases.png)


1. Reads are mapped against a database to find their most similar sequence - this is the database
2. They inherit the taxonomy of this matched reference - this is the taxonomy file 

Therefore, we always need two files: 

**The taxonomy database**
Contains the organism names and the phylogenetic hierarchy of each taxon. This means for a given species, it contains the information of its parent ranks as well as their taxomic identifiers

The default is the NCBI taxonomy database

Within the taxonomy file, there is a
- nodes.dmp file - represents the taxonomy nodes. It includes the TaxID of each node and the TaxID of the parent nodes
- names.dmp includes the scientific name of each TaxID

The workflow acceps as inputs: a folder, a tar.gz file, or a .zip file

If you wish to use a taxonomy database different from the one provided by the NCBI Taxonomy, you might want to use  to create NCBI-style taxdump files for custom taxonomy.


** The alignment database **

This can be Kraken 2 or MiniMap2

The Kraken2 database is used with the Kraken2 program for assigning taxonomic labels. Kraken2 uses a k-mer based approach which drastically reduces memory usage and time requirements.

It contains at least three files:

-hash.k2d: Contains the minimizer to taxon mappings.
-opts.k2d: Contains information about the options used to build the database.
-taxo.k2d: Contains taxonomy information used to build the database (the taxonomy database).


You will also need an extra file to run bracken. Bracken is a complementary tool to Kraken2 that uses the classifications to estimate the abundance of each organisms at a specific taxonomic rank.



The workflow acceps as input: a folder and a .tar.gz file, and importantly the taxonomy and database reference set must match in their labelling of sequences

# 2. Standard Kraken2 databases

Kraken2 supports a wide range of prebuilt databases that can be downloaded from . The selection includes databases of varying sizes and including different organisms. You can inspect the content of the database using the command kraken2-inspect (available with Kraken2). Larger databases contain more sequences and may provide more accurate results. On the other hand, though, they also require more computing resources (if you are worried about compute requirements, you can look into mitigating options like --kraken2_memory_mapping).

To download one of the databases (e.g. PlusPF-8), run:

`wget https://genome-idx.s3.amazonaws.com/kraken/k2_pluspf_08gb_20231009.tar.gz

These prebuilt databases also include the corresponding bracken files and are packaged as .tar.gz archives. So we need to uncompress the tar file

`tar -xzf k2_index.tar.gz -C .\k2_index\


To run the workflow with our database, and also provide the taxonomy database we specify:
`nextflow run wf-metagenomics/main.nf --database k2_index --taxonomy path_to_taxonomy_database

These prebuilt kraken2 databases use the NCBI taxonomy described in section above. You can download it with the next command. For other databases, you will need to find out which taxonomy database they use (see below for SILVA):



For the NCBI reference taxonomy database ,we install it using:
The taxonomy database
In this case, we will use the NCBI taxonomy, so we need to download it from , as it has been described in the Standard Kraken2 databases. 




CoPilot suggestion:

# Install Kraken2
`conda install -c bioconda kraken2

# Download the taxonomy
`kraken2-build --download-taxonomy --db $DBNAME

`kraken2-build --download-taxonomy <- this installs NCBI as default, or maybe use some sub-version, we don't need everything

# Download PR2 database
`wget https://github.com/pr2database/pr2database/releases/download/4.12.0/pr2_version_4.12.0_18S_mothur.tax.gz
`gunzip pr2_version_4.12.0_18S_mothur.tax.gz

# Add PR2 to the Kraken2 database
`kraken2-build --add-to-library pr2_version_4.12.0_18S_mothur.tax --db $DBNAME

# Build the database
`kraken2-build --build --db $DBNAME

# Install Bracken
`git clone https://github.com/jenniferlu717/Bracken.git
`cd Bracken
`make

# Generate a Bracken database
./bracken-build -d $DBNAME -k KMER_LEN -l READ_LEN

# Estimate the abundance with Bracken
./bracken -d $DBNAME -i $REPORT -o $OUTPUT -r READ_LEN

Replace $DBNAME with the path to your Kraken2 database, KMER_LEN with the k-mer length used for the Kraken2 database, READ_LEN with the average read length of your sequences, $REPORT with the path to the Kraken2 report file, and $OUTPUT with the path to the file where you want to store the Bracken results.

Then, you can use the following R script to read the Bracken output for further analysis:

# Load necessary library
library(tidyverse)

# Read the Bracken output file
bracken_data <- read_tsv("bracken_output.txt")

# View the first few rows of the data
head(bracken_data)

Replace "bracken_output.txt" with the path to your Bracken output file. The read_tsv function is part of the tidyverse package, which is a collection of R packages designed for data science.

Please note that these steps require a significant amount of disk space and RAM, depending on the size of the reference sequences you are including. Also, these are very basic usage of Kraken2, Bracken, and R. Depending on your specific needs, you might want to use additional options. You can find more information about the available options in the Kraken2, Bracken, and tidyverse manuals.



I think I can just use the standard taxonomy if i don't specify, but i just need to specify my Kraken2 database files (.k2d files hash, opts, taxo) and my bracken (Kmer_distrib) file

So if I can make my Kraken2 file and my bracken file I should be able to make this work for my 18S reads


To build my bracken file:

```
bracken-build -d kraken2_database/ -l 1000
generate_kmer_distribution.py \
    -i kraken2_database/database1000mers.kraken \
    -o database1000mers.kmer_distrib
```

For Kraken2 I need these arguments on wf-16S:

```
nextflow run ~/workflows/wf-metagenomics/main.nf \
    --fastq ~/workflows/wf-metagenomics/test_data/case01/ \
    --taxonomy NCBI_taxdump/ \
    --database kraken2_database \
    --bracken_dist kraken2_database/database1000mers.kmer_distrib
```



Let's just try build a kraken2 database for PR2


