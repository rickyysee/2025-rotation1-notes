# 6 Annotating Amino Acid Sequences
We added a lot of amino acid sequences to the original query by using a neighborhood + PLASS approach. We will be using `kofamscan` to annotate amino acid sequences.

`kofamscan` uses hidden markov models (HMM) from KEGG orthologs to assign KEGG ortholog numbers to new amino acid sequences.
![image](https://hackmd.io/_uploads/B1SrjCz6gx.png)
HMMs can give us weighted importance of amino acids. The above image shows a PFAM (protein family) HMM of the rpsG gene, which is highly conserved. The size of the amino acid depicts how likely we are to see that amino acid in that position. If there is no visible amino acid, that aa position is less important. This is flexible as it incorporates biological significance and works well on novel amino acid sequences.

`kofamscan` works with the KEGG pathways to assign KEGG orthologs to our aa sequences. This can tell us if any pathways are more complete in our neighborhood than our query.

### Running `kofamscan`
```
srun --account=ctbrowngrp -p bmh -J kofam -t 48:00:00 --mem=16G -c 6 --pty bash
```
`kofamscan` is not available as a conda package, but it is very useful so it's worth to install manually.

Let's create a whole environment for `kofamscan`. Then make a directory for it and link the `PLASS` result as well as the genomic protein sequences.
```
mamba create -n kofamscan hmmer parallel ruby kofamscan
conda activate kofamscan
cd ~/$(date +%Y)-$USER-rotation
mkdir -p kofamscan
cd kofamscan
ln -s ../plass/query_nbhd_plass.cdhit.fa ./
ln -s ../blast/GCA_001508995.1_ASM150899v1_protein.faa ./
```
Download the databases and executables for `kofamscan`. Then uncompress them.
```
wget ftp://ftp.genome.jp/pub/db/kofam/ko_list.gz                  # download the ko list 
wget ftp://ftp.genome.jp/pub/db/kofam/profiles.tar.gz           # download the hmm profiles
gunzip ko_list.gz
tar xf profiles.tar.gz
```
Make a config file to hold the `kofamscan` parameters.
```
# Path to your KO-HMM database
# A database can be a .hmm file, a .hal file or a directory in which
# .hmm files are. Omit the extension if it is .hal or .hmm file
profile: ./profiles

# Path to the KO list file
ko_list: ko_list

# Path to an executable file of hmmsearch
# You do not have to set this if it is in your $PATH
# hmmsearch: /usr/local/bin/hmmsearch

# Path to an executable file of GNU parallel
# You do not have to set this if it is in your $PATH
# parallel: /usr/local/bin/parallel

# Number of hmmsearch processes to be run parallelly
cpu: 6
```
Now run the program on our `PLASS` and the genomic aa sequence. (Try using `notify` since this will take long)
```
exec_annotation -f mapper --config config.yml -o query_nbhd_plass.clean_kofamscan.txt query_nbhd_plass.cdhit.fa
exec_annotation -f mapper --config config.yml -o GCA_001508995.1_ASM150899v1_protein_kofamscan.txt GCA_001508995.1_ASM150899v1_protein.faa
```
This took over 4 hours.

The output should look like this:
```
NORP88_1
NORP88_2
NORP88_3    K01869
NORP88_4
NORP88_5
NORP88_6    K07082
NORP88_7
NORP88_8    K07448
NORP88_9
NORP88_10   K07507
```
... but it doesn't...
:::spoiler nbhd output
0
1
4
5
9       K04567
11
12
15
18
23      K02358
:::
:::spoiler genome output
KUK63054.1
KUK63055.1      K07487
KUK63068.1
KUK63069.1
KUK63070.1
KUK63082.1
KUK63083.1
KUK63084.1
KUK63085.1
KUK63086.1      K17865
:::
**NOTE** [KEGGDecoder](https://github.com/bjtully/BioData/tree/master/KEGGDecoder) input needs the first column to delineate genomes. (i.e. genome_sample    KOnumber). The genome and sample should be separated by an underscore `_`.

Therefore, let's use `sed` to add a genome identifier to each file and then combine them into one.
```
sed 's/^/nbhd_/' query_nbhd_plass.clean_kofamscan.txt > nbhd-prepped.txt
sed 's/^/genome_/' GCA_001508995.1_ASM150899v1_protein_kofamscan.txt > genome-prepped.txt
cat *prepped.txt > kofamscan_results.txt
```
To run `KEGG-decoder` to visualize these results, let's make a new environment for its dependencies.
```
conda create -n keggdecoder python=3.6
conda activate keggdecoder
python3 -m pip install KEGGDecoder
```
Now run `KEGGDecoder` on this file we have created.
```
KEGG-decoder -i kofamscan_results.txt -o kegg_decoder_out --vizoption interactive
```
Use `scp` to download the file to your local computer and view the `.html` file in a browser!