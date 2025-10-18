# Taxonomic Discovery with Sourmash
We know the `SB1` sample contains bacteria and archaea but lets pretend we don't know this. We can approximate the species composition of our sample with `sourmash`.

Here is the tutorial on [sourmash](https://angus.readthedocs.io/en/2019/sourmash.html). It basically breaks our sequences into smaller pieces and looks for those pieces in databases.

### Prepare Data
Let's get a compute node.
```
srun -A ctbrowngrp -p bmh -J sourmash24 -t 24:00:00 --mem=16gb -c 1 --pty bash
```
Activate the environment and install sourmash.
```
conda activate dib-rotation
mamba install -y sourmash
```
move into the rotation directory and create a new `sourmash/` directory to hold output files.
```
cd ~/2025-$USER-rotation
mkdir -p sourmash
cd sourmash
```
In the `sourmash/` directory, create a symbolic link to the raw sequencing files.
```
ln -s ~/2025-$USER-rotation/raw-data/*fastq.gz ./
```
### Use `sourmash` on Raw Data
Create a signature, a compressed representation of the k-mers in a sequence with `sourmash sketch`.
```
sourmash sketch dna -o SRR1976948.sig \
 --name SRR1976948 \
 -p scaled=2000,k=21,k=31,k=51,abund \
 SRR1976948_*fastq.gz
```
To compare this signature to a database, use `sourmash gather`.

**Note** Visit [this page](https://sourmash.readthedocs.io/en/latest/databases.html) to see what databases are available to use with sourmash.

Download and uncompress the `sourmash` database.
```
curl -L https://osf.io/4f8n3/download -o genbank-k31.lca.json.gz
gunzip genbank-k31.lca.json.gz
```
Next, run `gather`, which will output to a `.csv` file and use the previously created `.sig` file against the `.json` database file.
```
sourmash gather -o SRR1976948-gather.csv \
 SRR1976948.sig genbank-k31.lca.json
```
**Output:**
* the name on the right is the name of the reference genome
* `overlap` is the number of distinct (unweighted) bases in the genome(s) in the sample file that match to the reference genome on that line
* `p_query` is what fraction of the data in the sample matches to that reference genome; **this is weighted**. This is an estimate of what fraction of the reads in the file would map to this reference.
* `p_match` is what fraction of the reference genome is hit at least once by the data in the sample; **this is unweighted**. Also known as "detection".
* **avg_abund** is the average abundance with which the matching portions of the reference genome are hit by data in the sample; **this is weighted**. Also known as "depth of sequencing".

**Note** `p_query` and `p_match` **do not** double-count k-mers or matches; in particular, you can sum across `p_query` for a metagenome without counting anything more than once.

`overlap` **does** count matches redundantly.
```
== This is sourmash version 4.8.3. ==                                           
== Please cite Brown and Irber (2016), doi:10.21105/joss.00027. ==              
                                                                                
selecting default query k=31.                                                   
loaded query: SRR1976948... (k=31, DNA)                                         
--                                                                              
loaded 93249 total signatures from 1 locations.                                 
after selecting signatures compatible with search, 93249 remain.                
                                                                                
Starting prefetch sweep across databases.                                       
Prefetch found 6296 signatures with overlap >= 50.0 kbp.                        
Doing gather to generate minimum metagenome cover.                              
                                                                                
overlap     p_query p_match avg_abund                                           
---------   ------- ------- ---------                                           
2.5 Mbp        0.6%   96.9%      19.4    LGHF01000260.1 Methanomicrobiales ar...
2.4 Mbp        0.4%   99.2%      13.3    LGGL01000207.1 Methanobacterium sp. ...
2.3 Mbp        2.4%  100.0%      76.7    LGGF01000001.1 Desulfotomaculum sp. ...
2.3 Mbp        0.4%  100.0%      14.0    LGFV01000060.1 Actinobacteria bacter...
2.2 Mbp        8.2%   97.7%     287.5    LGFZ01000006.1 Desulfotomaculum sp. ...
2.1 Mbp       12.4%   99.0%     452.6    LGFT01000012.1 Methanosaeta harundin...
2.0 Mbp        1.0%   99.0%      37.7    LGGY01000264.1 Marinimicrobia bacter...
1.9 Mbp        0.2%  100.0%       6.5    LGGC01000327.1 Bacteroidetes bacteri...
1.9 Mbp        0.2%   55.1%       7.4    MPPD01000289.1 Thermotogales bacteri...
1.9 Mbp        2.1%   99.5%      86.2    LGGX01000001.1 Candidate division TA...
1.8 Mbp        0.3%  100.0%      10.6    LGGM01000054.1 Clostridiales bacteri...
1.8 Mbp        1.8%   98.3%      78.5    LGHE01000021.1 Methanoculleus marisn...
1.6 Mbp        0.1%  100.0%       6.4    LGGR01000252.1 Petrotoga mobilis iso...
1.6 Mbp        0.2%   99.4%       8.3    LGGN01000313.1 Proteiniphilum acetat...
1.5 Mbp        0.3%  100.0%      15.2    LGGB01000030.1 Synergistales bacteri...
1.4 Mbp        1.3%  100.0%      69.3    LGGV01000121.1 Synergistales bacteri...
...
...
...
80.0 kbp       2.8%    1.8%    4159.8    FREM01000944.1 Enterococcus faecium ...
70.0 kbp       0.0%    2.7%      25.0    MWES01000001.1 Bacteroidetes bacteri...
found less than 50.0 kbp in common. => exiting

found 87 matches total;
the recovered matches hit 48.6% of the abundance-weighted query.
the recovered matches hit 5.4% of the query k-mers (unweighted).

WARNING: final scaled was 10000, vs query scaled of 2000

```

The percent matches are high because the authors **submitted metagenome-assembled genomes** into GenBank!

The second to last sentence mentions that only 5.4% of our sequences were captured by the database, which is common in metagenomes from rare environments.

### Bonus: Use `sourmash gather` on Trimmed Data

### Prepare Trimmed Data
Create a symbolic link to the data.
```
ln -s ../trim/SRR1976948_* ./
```
Create a signature with the trimmed data.
```
sourmash sketch dna -o SRR1976948-trim.sig \
 --name SRR1976948 \
 -p scaled=2000,k=21,k=31,k=51,abund \
 SRR1976948_*.trim.fastq.gz
```
Use `gather` between trimmed data and downloaded database.
```
sourmash gather -o SRR1976948-trim-gather.csv \
 SRR1976948-trim.sig genbank-k31.lca.json
```
> Can use `notify` command to send alert when this is done
::::spoiler Output
```
== This is sourmash version 4.8.3. ==                                             
== Please cite Brown and Irber (2016), doi:10.21105/joss.00027. ==                
                                                                                  
selecting default query k=31.                                                     
loaded query: SRR1976948... (k=31, DNA)                                           
--                                                                                
loaded 93249 total signatures from 1 locations.                                   
after selecting signatures compatible with search, 93249 remain.                  
                                                                                  
Starting prefetch sweep across databases.                                         
Prefetch found 6327 signatures with overlap >= 50.0 kbp.                          
Doing gather to generate minimum metagenome cover.                                
                                                                                  
overlap     p_query p_match avg_abund                                             
---------   ------- ------- ---------                                             
2.5 Mbp        0.7%   96.5%      17.3    LGHF01000260.1 Methanomicrobiales ar...  
2.4 Mbp        0.5%   99.2%      12.6    LGGL01000207.1 Methanobacterium sp. ...  
2.3 Mbp        2.7%  100.0%      70.4    LGGF01000001.1 Desulfotomaculum sp. ...  
2.3 Mbp        0.4%   99.6%      10.3    LGFV01000060.1 Actinobacteria bacter...  
2.2 Mbp        9.1%   97.7%     262.3    LGFZ01000006.1 Desulfotomaculum sp. ...  
2.1 Mbp       13.0%   99.0%     390.1    LGFT01000012.1 Methanosaeta harundin...  
2.0 Mbp        1.1%   99.0%      35.4    LGGY01000264.1 Marinimicrobia bacter...  
1.9 Mbp        0.2%  100.0%       6.2    LGGC01000327.1 Bacteroidetes bacteri...  
1.9 Mbp        2.6%   99.5%      84.9    LGGX01000001.1 Candidate division TA...  
1.8 Mbp        0.2%   53.7%       7.1    MPPD01000289.1 Thermotogales bacteri...  
1.8 Mbp        0.3%  100.0%      10.2    LGGM01000054.1 Clostridiales bacteri...  
1.8 Mbp        1.7%   98.3%      60.8    LGHE01000021.1 Methanoculleus marisn...  
1.6 Mbp        0.2%   99.4%       6.3    LGGR01000252.1 Petrotoga mobilis iso...  
1.6 Mbp        0.2%   99.4%       7.6    LGGN01000313.1 Proteiniphilum acetat...  
1.5 Mbp        0.3%   99.3%      14.0    LGGB01000030.1 Synergistales bacteri...  
1.4 Mbp        1.4%  100.0%      60.5    LGGV01000121.1 Synergistales bacteri...  
1.4 Mbp        0.2%   99.3%       9.8    LGFU01000166.1 Anaerolinea thermophi...  
1.4 Mbp        0.4%   88.1%      20.0    LGHA01000040.1 Anaerolinaceae bacter...  
1.4 Mbp        0.8%   95.0%      36.6    LGGW01000228.1 Mesotoga infera isola...  
1.1 Mbp        0.0%   36.2%       2.3    CP000252.1 Syntrophus aciditrophicus...  
1.1 Mbp        0.2%  100.0%       9.3    LGGO01000196.1 Candidate division WS...  
...
...
...
120.0 kbp      0.0%    1.0%       1.2    CP007509.1 Pseudomonas stutzeri stra...  
80.0 kbp       0.0%    1.1%       1.2    CP002881.1 Pseudomonas stutzeri stra...  
70.0 kbp       0.0%    2.7%      24.8    MWES01000001.1 Bacteroidetes bacteri...  
50.0 kbp       0.0%   71.5%       5.4    LGHI01000016.1 Thermoplasmatales arc...  
50.0 kbp       0.0%    1.4%       1.2    MIAV01000001.1 Spirochaetes bacteriu...  
found less than 50.0 kbp in common. => exiting                                    

found 84 matches total;                  
the recovered matches hit 50.9% of the abundance-weighted query.                  
the recovered matches hit 12.2% of the query k-mers (unweighted).                 

WARNING: final scaled was 10000, vs query scaled of 2000 
```
::::
Seems like we were able to increase the amount of queries that matched to 12.2%.
