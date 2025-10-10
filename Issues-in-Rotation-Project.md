# Rotation Notes/Issues
## 0 Setup Workspace
* Change all instances of hard-coded directory name for rotation project to `~/$(date +%Y)-$USER-rotation/`
    * This will automatically fill in current year and username
* After loading conda but before downloading fastq files, avoid running out of disk space in home
    * Make directory in `/group/ctbrowngrp6/$(date +%Y)-$USER-rotation` (Titus's disk space; check if there is a more recent directory such as `ctbrowngrp7`)
    * Make symbolic links in your home `~/` directory
    ```
    ln -s /group/ctbrowngrp6/$(date +%Y)-$USER-rotation ~/
    mv ~/.conda ~/$(date +%Y)-$USER-rotation
    ln -s ~/$(date +%Y)-$USER-rotation/.conda ~/
    ```
* In `srun` commands that use partitions other than those found in `adamgrp`, must specify `--account=ctbrowngrp`
## 1 Finding and Downloading Data
* Use `md5sum` to compute the md5 hash of any downloaded files and ensure they match the database they were downloaded from
## 2 Quality Control
## 3 Taxonomic Discovery
* For `sourmash sketch` command, you need to add `dna` before the `-o` flag in order to specify what type of signature you want to create.
## 4 Bin Completion
* Paper used to illustrate use of cDBG is not explicitly linked
* Link the spacegraphcats github repo
* `pip install -e ./spacegraphcats/` since no longer a beta?
## 5 Assembling Neighborhood
## 6 Annotating Amino Acid Sequences
* `KEGGDecoder` install is deprecated, instead, create a new environment with python=3.6. ([source](https://github.com/bjtully/BioData/tree/master/KEGGDecoder))
    ```
    conda create -n keggdecoder python=3.6
    conda activate keggdecoder
    python3 -m pip install KEGGDecoder
    ```
    > Note this makes a whole new environment that no longer has `kofamscan`
* For some reason, the files I created after `kofamscan` did not have characteristic `genome_sample` naming format for rows. I checked previous files (raw reads) and it seems like they also did not have this naming scheme. To follow this naming scheme (which `KEGGDecoder` requires), you can use `sed` to add in some genome names.
    ```
    sed 's/^/nbhd_/' query_nbhd_plass.clean_kofamscan.txt > nbhd-prepped.txt
    sed 's/^/genome_/' GCA_001508995.1_ASM150899v1_protein_kofamscan.txt > genome-prepped.txt
    cat *prepped.txt > kofamscan_results.txt
    ```