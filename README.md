# Singularity for reproducible research and collaboration



## What containerisation is and when it might be useful 

* complex dependencies
* Different OS to running system
* Contained toolsets to give to others
* If it works on your laptop, it'll work everywhere - caveats!
    * If it contains code compiled for specific CPU instruction sets then it will only work on CPUs with those instructions.
    * Eg AVX-512 instructions speed up CPU machine learning but are only available on very new Intel processors etc.
* Reproducable research - give others your exact version and setup.
* Docker on HPC
    


## How containerisation can support reproducible and collaborative research 

* can share the container image - but it could be large
    * So maybe share the definition file instead!
* base off an existing docker image or singularity image.
* Your exact code and enviroment 

## How to use Singularity to perform a computing task, such as run an R job, or part of a genomics workflow 

Start simple, locally run the lolcow example.

```bash
# Note no local file, runs from your local cache
singularity pull shub://GodloveD/lolcow #

#can also use build - converts to latest singularity image formate after downloading it.
singularity build mylolcow.sif shub://GodloveD/lolcow

#run container with it's internal runscript - more later when we build our own.
singularity run lolcow_latest.sif
#or
./lolcow_latest.sif

```

How about something more useful. Tensorflow convolutional MNIST example
```bash
git clone https://github.com/andre-geldenhuis/singularity_webinar_examples

# pull the singlarity image - not this creates a local file.
singularity pull docker://tensorflow/tensorflow:latest-gpu-py3

#--nv for GPU
singularity exec --nv tensorflow_latest-gpu-py3.sif \
    python3 #exec -run arbitrary command in container.
#amusingly this is slower on GPU than a reasonable CPU due to setup costs
```

An Rshiny example
```bash 
# a simple example container with shiny.  Definition file here: https://singularity-hub.org/containers/1788
singularity pull shub://jekriske/shinyapp

#open shiny in a local firefox window
singularity run shinyapp_latest.sif
```

Genomics example, maxbin2
```bash
#find the container tags on https://bioconda.github.io/recipes/maxbin2/README.html   sadly lastest doesn't work
singularity build maxbin2.sif docker://quay.io/biocontainers/maxbin2:2.2.7--he1b5a44_1

#get some raw data - though currently downloads.jbei is down
mkdir rawdata
curl https://downloads.jbei.org/data/microbial_communities/MaxBin/getfile.php?20x.scaffold > rawdata/20x.scaffold
curl https://downloads.jbei.org/data/microbial_communities/MaxBin/getfile.php?20x.abund > rawdata/20x.abund

mkdir output


#run maxbin with 4 threads
singularity exec maxbin2.sif run_MaxBin.pl -contig rawdata/20x.scaffold -abund rawdata/20x.abund -out output/20x.out -thread 4
``` 

```bash
singularity shell maxbin2.sif

#try R
R  
```

```R
R.version # Probably different than your base system
quit()
```

## How to build a custom container for your workflow 

Sandbox - you can share this with others, but it's better to use this to figure out how to build a more complex .def file.

example.def
```bash
BootStrap: library
From: ubuntu:20.04

%post
apt-get update && apt-get -y install wget build-essential 

%runscript
    exec echo "$@" #Just print anything sent to the container to the terminal

%labels
    Author Andre
```
Now build it

```bash
sudo singularity build --sandbox my_example/ example.def
```

Get inside the container in writable mode and try update the packages
```bash
singularity shell my_example/ example.def

apt update  #fails, let's try writable mode
exit

sudo singularity shell --writable my_example/ example.def
apt update
apt install sqlite3
```

A more complex example, let's build it up

idba_start.def
```bash
BootStrap: library
From: ubuntu:20.04

%post
apt-get update && apt-get -y install wget build-essential 

%runscript
    exec "$@"  #run anything given to the container as a command
               #./idba_start.img idba  will run the command idba 
               # inside the container

%labels
    Author Andre
```

Install miniconda and get idba going, then put all that stuff into the post section

```bash
sudo singularity shell --writable idba/
```

```bash
#Download the minconda installer
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
# verify the hash, should be 
# 957d2f0f0701c3d1335e3b39f235d197837ad69a944fa6f5d8ad2c686b69df3b
#Verying on 
sha256sum Miniconda3-latest-Linux-x86_64.sh

#Silently install miniconda to /opt/miniconda
bash Miniconda3-latest-Linux-x86_64.sh -b -p /opt/miniconda

# Install idba, we don't need to activate the enviroment, we can just do it directly as we're installing into base
/opt/miniconda/bin/conda install -y -c bioconda idba

#See if idba is installed by looking in the miniconda base bin folder

ls /opt/miniconda/bin | grep idba

```

Now we could create an immutable image ready for use on the HPC or sharing with a collegue via ```sudo singularity build idba.sif idba/```. However a better approach is to create a more detailed ```.def``` file that contains all the steps done above.  That way we can share just the small .def file and someone can recreate our workflow, plus it's good to be able to come back to your work in 6 months and have living documentation about what you did exactly.

idba_dafe.def
```bash
BootStrap: library
From: ubuntu:20.04

%post
apt-get update && apt-get -y install wget build-essential 
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

# check the correct version of the miniconda installer has been downloaded
echo "**************"
echo "Does the following has match that of the linux installer on the miniconda page?"
echo "https://docs.conda.io/en/latest/miniconda.html#linux-installers"
sha256sum Miniconda3-latest-Linux-x86_64.sh

# Silently install miniconda to /opt/miniconda
bash Miniconda3-latest-Linux-x86_64.sh -b -p /opt/miniconda

# Install bioconda in base enviroment. -y forces install without asking if you're sure.
/opt/miniconda/bin/conda install -y -c bioconda idba

%environment
    PATH=$PATH:/opt/miniconda/bin/
    # if you had installed idba into a enviroment, eg myenv, then you would add that to the path too. like:
    #PATH=$PATH:/opt/miniconda/bin/:/opt/miniconda/envs/myenv/bin/

%runscript
    exec "$@"

%labels
    Author Andre
```

Now build the image.  Note that the sha256 check is still manual and get's lost in the cluster.  If someone has a working better idea, please feel free to submit a pull request against this repo.

```bash
sudo singularity build idba_safe.img idba_safe.def
```




## How to run a container on an HPC environment such as Rāpoi

Let run the container on the HPC. (Note you could also just download the data locally and run it there)

Copy the file idba_safe.img to raapoi using sftp etc.

In your project folder, download the data
```bash
mkdir data
cd data 
wget --content-disposition goo.gl/JDJTaz #sequence data
wget --content-disposition goo.gl/tt9fsn #sequence data
cd ..  #back to our project director
```

The reads we have are paired end fastq files but idba requires a fasta file. We can use a tool built into our container to convert them. We'll do this on the raapoi login node as it is a fast task that doesn't need many resources.

```bash
module load singularity
singularity exec idba_full.img fq2fa --merge --filter data/MiSeq_Ecoli_MG1655_50x_R1.fastq data/MiSeq_Ecoli_MG1655_50x_R2.fastq data/read.fa
```

We need a sbatch submission script to run our job on a node
submit_idba.sh
```bash
#!/bin/bash

#SBATCH --job-name=idba_test
#SBATCH -o output.out
#SBATCH -e output.err
#SBATCH --time=00:10:00
#SBATCH --partition=quicktest
#SBATCH --ntasks=12
#SBATCH --mem=1G

module load singularity

singularity exec idba_full.img idba idba_ud -r data/read.fa -o output
```

Submit our job with
```bash
sbatch submit_idba.sh
```
