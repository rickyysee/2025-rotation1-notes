# Environments and Packages
### Metagenomics (Rotation)
Created `.conda/` in Titus's disk space and made a `ln` to it in my home directory.
```
module load conda
conda activate <environment-name>
```
### Key Packages in each environment
#### `dib-rotation`
```
mamba install -y sourmash # TBD
mamba install -y fastqc # Used for looking at read quality
mamba install -y fastp # Used for quality trimming
mamba install -y khmer # Used for k-mer trimming
mamba install plass # protein-level assembly
mamba install blast # sequence search/alignment
mamba install -y snakemake-minimal # workflow system
mamba install graphviz # visualize DAGs
```
> Should I move this to `scratch3` and how?
[Solution](#Moving-Files-to-Shared-Disk-Space)
#### `kofamscan`
```
mamba create -n kofamscan hmmer parallel ruby kofamscan
conda activate kofamscan
```
#### `keggdecoder`
```
conda create -n keggdecoder python=3.6
conda activate keggdecoder
python3 -m pip install KEGGDecoder
```
#### `spacegraphcats`
```
cd ~
git clone https://github.com/spacegraphcats/spacegraphcats/
conda env create -f spacegraphcats/environment.yml -n sgc
conda activate sgc
pip install -e ~/spacegraphcats/
```
#### `renv`
```
mamba create -n renv r-dplyr=0.8.3 bioconductor-biostrings=2.54.0 
conda activate renv
R
```
### GGG 201A
Created a conda environment in Titus's `scratch3` directory called `fastqc` and made a symbolic link to it in my home directory.
```
mkdir -p ~ctbrown/scratch3/2025-conda/$USER
conda create -p ~ctbrown/scratch3/2025-conda/$USER/fastqc \
    -y fastqc
mkdir -p ~/.conda/envs
ln -s ~ctbrown/scratch3/2025-conda/$USER/fastqc ~/.conda/envs/
conda activate fastqc
```
### Moving Files to Shared Disk Space
In order to avoid using the 20GB in my home directory, I created a directory in Titus's shared disk space following the `YEAR-user-project` nomenclature. I then made a symbolic link to it in my home directory.
```
mkdir /group/ctbrowngrp6/2025-rcantua-rotation
ln -s /group/ctbrowngrp6/2025-rcantua-rotation ~/
```
Then I moved the `2025_rotation` directory to the new harddrive.
```
mv ~/2025_rotation ~/2025-rcantua-rotation
```
"Unpack" `2025_rotation` so its not a folder within a folder
```
mv ~/2025-rcantua-rotation/* ~/2025-rcantua-rotation
cd ~/2025-rcantua-rotation
rm -r 2025_rotation
```
I also moved the `~/.conda/` directory (which holds the environments) to the new harddrive and made a symbolic link.
```
mv ~/.conda ~/2025-rcantua-rotation
ln -s ~/2025-rcantua-rotation/.conda ~/
```
Can also use `conda clean -a` to make conda remove temporary files and "clean up after itself".

### Notifications from HPC
I have adapted some code from Titus's notify script to give you a function that runs a command and notifies you when it completes.

The following should be copy-pasted into your `~/.bashrc` or `~/.bash_profile` file.
```
# Modified from Titus's code
notif() {
    DETAILS=$(history | tail -1)
    cat <<EOF | sendmail $USER@ucdavis.edu
Subject: Job Finished
From: Farm Notification <$USER@farm.cse.ucdavis.edu>

Command: $DETAILS

Start: $TMP     
End: $(date)"
EOF
}

notify() {
    TMP=$(date)
    "$@" && notif 
}
```
* `~/.bashrc` is sourced first, `~/.bash_profile` is sourced after `srun` (either can also be manually sourced)
* In `~/.bashrc`, your bigmem node will automatically source it
* In `~/.bash_profile`, the head node will automatically source it

#### Usage
```
notify COMMAND -OPTIONS ARGUMENTS
```
The `notify` function will run `notif` **after** your chosen command has completed, which will then use `sendmail` (which seems to be automatically installed in the HPC?) to email your UCD email.

The email will consist of the command name `$DETAILS`, start `$TMP`, and end `$(date)` time.

### Note to Improve
Will try to figure out how to add the `usr/bin/time -v` command to run right before your chosen command every time and show the output in the email?

**MAYBE**
Still need to test this more since it redirects `stderr` to a variable.
```
notif() {
    DETAILS=$(history | tail -1)
    cat <<EOF | sendmail "$USER@ucdavis.edu"
Subject: Job Finished
From: Farm Notification <$USER@farm.cse.ucdavis.edu>

Command: $DETAILS

Error/Resources:
$1

Timestamp: $(date)
EOF
}


notify() {
    TMP=$(mktemp)
    # Run command with /usr/bin/time, letting command output go normally,
    # while capturing only stderr in TMP
    /usr/bin/time -v "$@" 2> "$TMP"
    USAGE=$(<"$TMP")
    rm -f "$TMP"
    notif "$USAGE"
}
```