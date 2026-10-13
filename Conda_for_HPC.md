# Conda on HPC

This Markdown document was created by Patrick M. Hooper on 13/02/2024.

# What is Conda?

1. Conda is a package and environment manager that is these days the easiest way to install most bioinformatics tools. 
2. Conda handles different environments very nicely making life easier. e.g. sometimes software A has dependencies which include software B and software C, but then software C will depend on a different type of software B. Conda allows us to create and manage separate environments to avoid these types of version conflicts.
3. Conda helps with reproducibility. You don't need a specific version of packages to make some script work.

## 1. Installing Conda

[This information comes from AstroBioMike's website](https://astrobiomike.github.io/unix/conda-intro#binder-available) 

Conda comes in two broad forms: *Anaconda* and *Miniconda*. Anaconda is large with lots of programs already installed. Miniconda is more lightweight, meaning we can install just what we want. This is preferable to not clog up our home directory!

The download page for Miniconda is [here](https://docs.conda.io/projects/miniconda/en/latest/)

For the server, we want the Linux version as the server runs in Linux, even if you're logging in from a Windows PC etc.

##Downloading Conda:

1. Navigate to your home directory
2. Download miniconda directly to the folder:

`wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh`

3. Make the file executable:

`chmod +x Miniconda3-latest-Linux-x86_64.sh`

4. Run this file to install miniconda:

`bash ./Miniconda3-latest-Linux-x86_64.sh

5. Answer 'yes' to any questions, accept default options, and wait for installation to complete. Once done, *restart your terminal*. You should now see base next to your command line: "(base) patrh5@hpc-head-002:~$"

**NOTE: Python is also installed as part of base Conda, but R is not!**

## 2. Creating and navigating environments

Environments represent the infrastructure our computer is currently operating in. This includes like if certain variables exist and how they are set. 

**Base environment**  
The "base" conda environment is the home base inside conda. We wouldn't want to install lots of complicated programs here, as the more things added, the more likely something is going to end up having a conflict. But the base environment is somehwere we might want to install simpler programmes we tend to use a lot. 

**Making a new environment**  
The simplest way to create a new conda environment is like so:

`conda create -n new-env` '-n' specifying the environment name

**Activating / entering our new environment**

`conda activate new-env` You should now see the prompt has changed to "(new-env) patrh5@hpc-head-002:~$"

**List all your environments**
`conda env list

**Deactivating / leaving our new environment**
`conda deactivate` You should now see "base" next to your login once again

**Remove an environment

`conda env remove -n python-v2.7

### Specifying package versions within a Conda environment

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

**IMPORTANT: The safest way to install is to specify your channels:**

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

**IMPORTANT - DO THIS IN YOUR DESIRED CONDA ENVIRONMENT, IT'S NOT ADVISABLE TO MAKE THESE CHANNELS IN YOUR HOME DIRECTORY**

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
