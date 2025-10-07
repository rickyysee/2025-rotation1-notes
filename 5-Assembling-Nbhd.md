# 5 Assembling a Nbhd (neighborhood)
Let's make our query neighborhood and see if we have any additional sequence that wasn't originally there. Nucleotide assembly *de novo* is hard. Let's assemble at the amino acid level.

We will use `PLASS` (protein-level assembly) to assemble our query neighborhood. Then we will compare proteins we assembled with `PLASS` to the original query.

### Running `PLASS` and Formatting Output
Let's get a compute node.
```
srun --account=ctbrowngrp -p bmh -J plass -t 24:00:00 --mem=16G -c 1 --pty bash
```
Install `PLASS` into `dib-rotation` environment.
```
conda activate dib_rotation
mamba install plass
```
Now run `PLASS`.
```
cd ~/$(date +%Y)-$USER-rotation
mkdir -p plass
cd plass
plass assemble ../sgc/SRR1976948_k31_r1_search_oh0/GCA_001508995.1_ASM150899v1_genomic.fna.gz.cdbg_ids.reads.gz query_nbhd_plass.fa tmp
```
We have to do some reformatting of the `PLASS` output. `PLASS` shows `*` for STOP codons. Let's download a script to remove these.
```
wget https://raw.githubusercontent.com/spacegraphcats/2018-paper-spacegraphcats/master/pipeline-base/scripts/remove-stop-plass.py
python remove-stop-plass.py query_nbhd_plass.fa
```
`PLASS` also doesn't condense redundant amino acid sequences if they came from slightly different nucleotides.

We can use `CD-HIT` to cluster sequences at 100% identity and use a representative sequence instead.
```
mamba install cd-hit
cd-hit -c 1 -i  query_nbhd_plass.fa.nostop.fa -o  query_nbhd_plass.cdhit.fa
```
### Compare AAs in in our Neighborhood to those in our Query
We could use `sourmash sketch protein` to generate signatures of our amino acid fasta files, then use `sourmash compare` to compare them. Let's do something different than `sourmash` to get a more detailed answer.

Let's use `BLAST` to compare all protein sequences in our `PLASS` neighborhood to the ones in the GenBank assembly from the paper. Let's download the protein sequence from GenBank into a `blast/` folder
```
cd ~/$(date +%Y)-$USER-rotation
mkdir -p blast
cd blast
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/001/508/995/GCA_001508995.1_ASM150899v1/GCA_001508995.1_ASM150899v1_protein.faa.gz
gunzip GCA_001508995.1_ASM150899v1_protein.faa.gz
```
Let's link the neighborhood AA sequences into this `blast/` folder. Also, install `blast`
```
ln -s ../plass/query_nbhd_plass.cdhit.fa ./
mamba install blast
```
Now we can use BLAST. `blastp` will compare protein sequence to protein sequences. We will use `-query / -subject` format since we are not comparing with a database. We will use the flag `-outfmt 6` to make a tab-deliminated format output for use with R later.
R output
```
blastp -query query_nbhd_plass.cdhit.fa -subject GCA_001508995.1_ASM150899v1_protein.faa -outfmt 6 -out query_nbhd_blast.tab
```
Let's set up an R environment with `dplyr` and `Biostrings`. `dplyr` is useful for data manipulation and `Biostrings` makes reading amino acid sequences simple.
```
mamba create -n renv r-dplyr=0.8.3 bioconductor-biostrings=2.54.0 
conda activate renv
```
Start R by running
```
R
```
Run the follwing code in R to analyze the BLAST results.
```
library(Biostrings)  # import biostrings functions into current environment
library(dplyr)       # import dplyr functions into current environment

nbhd <- readAAStringSet("query_nbhd_plass.cdhit.fa") # import the plass nbhd
nbhd_aas <- length(nbhd)                                # get number of AAs in nbhd
blast <- read.table("query_nbhd_blast.tab")             # import blast results
query_aas <- length(readAAStringSet("GCA_001508995.1_ASM150899v1_protein.faa"))

blast_100 <- filter(blast, V3 == 100)    # retain only AAs that were 100%
aas_100 <- length(unique(blast_100$V2))  # count num aas 100% contained

aas_100/nbhd_aas # calculate the percent of AAs from the nbhd that were in the query
aas_100/query_aas # calculate the percent of query that was in the nbhd
```

:::spoiler Output
```
[1] 0.01514226 # percent of AAs from nbhd that were in query
[1] 0.7080689 # percent of query in the nbhd
```
:::
**Ask Titus**
What does this mean? Only 1.5% of the sequences in the nbhd are in the query, and 70% of the sequences in the qeury are in the nbhd. It seems there are many amino acid sequences that neighborhood analysis included, and even some that it removed??

To quit R, type `quit()`