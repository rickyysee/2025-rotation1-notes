# Bin Completion with `spacegraphcats`
There are 5 main ways to gain for information from metagenome samples.

**1. Align reads to genomes that `sourmash` matched**
`sourmash` matches with k-mers, which can be very specific especially with 31-mers. Can instead align reads with close relatives and maybe increase the amount of reads being classified.

**2. *de novo* assemble and bin the reads into metagenome-assembled genomes**
This does not require a reference and tries to make genomes (bins) from the acquired metagenome data. This works by finding overlaps between reads and **assembling** "contiguous sequences" (contigs). These contigs can vary in size and depend on depth, coverage, and biological properties of sample. Often, the contigs are **binned** into metagenome-assembled genomes depending on tetranucleotide frequency and abundance information. The frequency of tetranucleotides in a genome is usually conserved, which can be exploited. Additionally, if two contigs belong together, they probably have similar abundance.
*Limitations*
If there is low coverage and/or many viable assembly       options, this can fail. This can be due to errors or       high strain variation. Tetranucleotide frequency is more reliable when contigs are >2000 bp, which results in many contigs not binning.
> This is what the original paper did.

**3. Continue the analysis at the gene level**
Often, we can assemble more contigs than we can bin. You can annotate open reading frames (ORFs) on contigs. This can give you an idea of functional output of metagenome.

**4. Continue the analysis with read-based techniques**
If there is no reference and the reads don't bin or assemble, there are other tools to estimate taxonomy or function. Some are `Kraken` and `mifaser`.

**5. Exploit connectivity of DNA to assign more reads to pangenome of `sourmash` matches**
Pangenome is collection of genetic material for a given species/strain in a population. `spacegraphcats` can accomplish this.

### Representing all the k-mers: de Bruijn and compact de Bruijn graphs
In order to try to piece reads together, we need to assess all potential overlaps between reads. de Bruijn graphs represent the possible overlaps. You first chop a sequence into k-mers and find all overlaps between these k-mers of size k-1. After, you connect k-mers (nodes) together based on these overlaps.
![image](https://hackmd.io/_uploads/BkP2yj16ee.png)
A compact de Bruijn graph (cDBG) collapses any linear paths into a single node since there is only one possible path.
![image](https://hackmd.io/_uploads/rJhyloJTgx.png)
With a large enough k-mer size (roughly 31), k-mers originating from the same organism tend to connect together since they were in the orignal genome. All k-mers in the metagenome will be represented in a cDBG.

### cDBG in the Paper
```
Excerpt from Unknown Abstract
Dissimilatory perchlorate reduction is an anaerobic respiratory pathway that in communities might 
be influenced by metabolic interactions. Because the genes for perchlorate reduction are horizontally 
transferred, previous studies have been unable to identify uncultivated perchlorate-reducing 
populations. Here we recovered metagenome-assembled genomes from perchlorate-reducing sediment 
enrichments and employed a manual scaffolding approach to reconstruct gene clusters for perchlorate 
reduction found within mobile genetic elements. 
```
The authors knew that dissimilatory perchlorate reduction is important in these microbial communities but are hard to identify due to horizontal gene transfer. Strain variation likely impeded assembly due to differences in similar genes across organisms. They knew the gene *should* be present, so they found it in the cDBG and saw flanking sequences from multiple species. In different conditions (salinity), the gene is flanked by different amounts of organisms.
![image](https://hackmd.io/_uploads/SyZ-7okplg.png)

### Scaling up
Previous study created a cDBG for each sample, visualized with `Bandage`, then used BLAST to find genes of interest. This is time-consuming is mostly manual. This approach also does not scale since the complexity of the cDBG causes queries to be computationally intensive. Imagine if this could work for *every* gene assembled in a metagenome. Most bins are not 100% complete, but what if we used the bin as a query and pull out the context within a cDBG. Perhaps this could complete the bin by pulling the sequence context.

`spacegraphcats` uses a novel data structure to represent cDBG while maintaining biologica relationships. It also uses novel algorithms to efficiently query the cDBG.
![image](https://hackmd.io/_uploads/Hkrsro16gg.png)
The left is a representative cDBG while the right shows the same cDBG after simplified with spacegraphcats.

Spacegraphcats queries work by decomposing the query into k-mers, finding the node in which a query k-mer is contained within the spacegraphcats graph, and returning all of the k-mers in that node. This process is efficient enough to work on the whole metagenome for every k-mer in the query.

### Running `spacegraphcats`
We will start a new environment for this package.
```
srun -p bmh -J sgc -t 48:00:00 --mem=70gb -c 2 --pty bash
```
Go to your home directory, clone the GitHub repo for `spacegraphcats`, then use the `.yml` file to create an environment with required packages.
```
cd ~
git clone https://github.com/spacegraphcats/spacegraphcats/
conda env create -f spacegraphcats/environment.yml -n sgc
```
**Note** This took a long time for me and returned a SafetyError, will try to proceed.
```
SafetyError: The package for typing_extensions located at /cvmfs/hpc.ucdavis.edu/sw/conda/pkgs/typing_extensions-4.12.2-pyha770c72_0                            
appears to be corrupted. The path 'site-packages/typing_extensions-4.12.2.dist-info/RECORD'      
has an incorrect size.                  
                  
  reported size: 777 bytes              
                  
  actual size: 517 bytes 
```
When activating environment, received no error :-)
```
conda activate sgc
```
Install `spacegraphcats` in sgc environment with `pip`.
```
pip install -e ~/spacegraphcats/
```
Time to decide what query to use. Let's try *Desulfotomaculum sp. 46_80*, which was 100% present in the metagenome.

Search the [NCBI taxonomy browser](https://www.ncbi.nlm.nih.gov/Taxonomy/Browser/wwwtax.cgi) for this genome. Click the link to the *BioSample* repository and click on *Assembly*. Navigate to the GenBank assembly [ASM150899v1](https://www.ncbi.nlm.nih.gov/datasets/genome/GCA_001508995.1/). Finally, access the FTP browser. We want to download the `GCA_001508995.1_ASM150899v1_genomic.fna.gz` file.
```
cd ~/$(date +%Y)-$USER-rotation
mkdir -p sgc
cd sgc
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/001/508/995/GCA_001508995.1_ASM150899v1/GCA_001508995.1_ASM150899v1_genomic.fna.gz
```
Now we need to make a `conf1.yml` file that will hold the parameters for our query. Make a file with `nano`.
```
nano conf1.yml
```
Paste this into the file and save it with `Ctrl+x`, `y`.
```
catlas_base: 'SRR1976948'
input_sequences:
- ../abundtrim/SRR1976948.abundtrim.fq.gz
ksize: 31
radius: 1
search:
- GCA_001508995.1_ASM150899v1_genomic.fna.gz
searchquick: GCA_001508995.1_ASM150899v1_genomic.fna.gz
```
> * `catlas_base` - name of the output database
> * `input_sequences` - file(s) to use to make k-mers
> * `ksize` - k-mer size
> * `radius` - size at which to perform queries??
> * `search` - genome to query against

Now run `spacegraphcats`.
```
python -m spacegraphcats run conf1.yml extract_contigs extract_reads --nolock 
```
This will take a **long time** to run. This will produce 3 directories:
* `SRR1976948` - contains the cDBG
* `SRR1976948_k31_r1` - contains spacegraphcats data structs
* `SRR1976948_k31_r1_search_oh0` - output of query, including contigs (single paths) from cDBG, reads that contain k-mers in those contigs, and sourmash signature from the contigs.

### Compare Query to Neighborhood
Let's compare the query to the genome we downloaded.

First make a signature of the genome with `sourmash`. Our previous command already created a signature of the query so we can then compare the two.
```
sourmash sketch dna -p k=21,k=31,k=51,scaled=2000 -o GCA_001508995.1_ASM150899v1_genomic.sig GCA_001508995.1_ASM150899v1_genomic.fna.gz
sourmash compare -k 31 --csv comp1.csv *sig SRR1976948_k31_r1_search_oh0/*sig
```

:::spoiler Output
```
== This is sourmash version 4.8.3. ==
== Please cite Brown and Irber (2016), doi:10.21105/joss.00027. ==

loading 'SRR1976948_k31_r1_search_oh0/GCA_001508995.1_ASM150899v1_genomic.fna.gz
loaded 2 signatures total.
NOTE: downsampling to scaled value of 2000

0-GCA_001508995.1...    [1.    0.776]
1-nbhd:LGGF010000...    [0.776 1.   ]
min similarity in matrix: 0.776
```
:::

:::spoiler Original `gather`
```
2.2 Mbp        9.1%   97.7%     262.3    LGFZ01000006.1 Desulfotomaculum sp. ...
```
:::
With the cDBG, we found 77.6% similarity between query and reference.