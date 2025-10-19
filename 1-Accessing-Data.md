# Accessing Data

### Finding and downloading data
We will be working with previously published data of a metagenome sample from an Alaskan oil reservoir from [this paper](https://journals.asm.org/doi/10.1128/mbio.01669-15).

The paper provides accession numbers on page 9.

:::info
***Nucleotide sequence accession numbers.** This whole-genome shotgun project has been deposited at DDBJ/EMBL/GenBank under the accession numbers LDZP00000000 to LDZU00000000 and LGEN00000000 to LGHI00000000. The version described in this paper is version LDZP00000000 to LDZU00000000 and LGEN00000000 to LGHI00000000. The raw reads have been deposited at DDBJ/EMBL/GenBank under the accession number [SRP057267](https://www.ncbi.nlm.nih.gov/sra/?term=SRP057267). The project description and related metadata are accessible through BioProject number [PRJNA278302](http://www.ncbi.nlm.nih.gov/bioproject/PRJNA278302/).*
:::
Navigate to GenBank to see what the files look like. We are interested in the last sample,`metagenome SB1 from not soured petroleum reservoir, Schrader bluff formation, Alaskan North Slope`
![image](https://hackmd.io/_uploads/ryv_46-hxx.png)

If we click the run link `SRR1976948`, this will take us to the SRA Run Browser. We can also search the European Nucleotide Archive (ENA) for this accession number.

### Downloading Sequencing Data
This rotation project will exist in the `~/2025_rotation` folder on the FARM cluster.

Make a `/raw_data` folder and use `wget` to download the sequence files with their accession numbers.
```
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR197/008/SRR1976948/SRR1976948_1.fastq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR197/008/SRR1976948/SRR1976948_2.fastq.gz
```
:::spoiler Old issue with tmux
**Note**
Tried using `tmux` along with `&` to have these downloads run in the background while I went to the gym, but the jobs ended prematurely (~21%).

Will instead try to do a `tmux` session, then use:
```
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR197/008/SRR1976948/SRR1976948_1.fastq.gz ; wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR197/008/SRR1976948/SRR1976948_2.fastq.gz
```
> This will run one command after the other.

Then detach from the session.
:::
### FASTQ Format
:::info
The fastq format is a file system used for DNA sequences.

Line 1. Always begins with '@' and information about the read
Line 2. Actual DNA sequence
Line 3. Always begins with a '+' and sometimes the same info in line 1
Line 4. Has a string of characters that represents quality scores for DNA sequence, must be same size as line 2
::: 
We can observe the first few lines of files with `head`. For these sequences, we must first use `zcat` to temporarily unzip the file then pipe `|` it through to `head`.
### Quality
:::info 
The 4th line shows quality scores for respective bases based on base call accuracy. Quality scores are single characters that are encoded depending on the sequencing machine used. This machine used standard Sanger quality PHRED scoring, using Illumina version 1.8 onwards. 
```
Quality encoding: !"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJ
                   |         |         |         |         |
Quality score:    01........11........21........31........41
```

Higher scores mean we are more confident the base is correct, but this scale is logarithmic. A 10 means an accuracy of 90% whereas a score of 20 is an accuracy of 99%. These are calculated from the base calling algorithm and depend on how much signal was captured.
:::
### Assessing Quality with FASTQC
[FastQC](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/) is a program that can help visualize the quality of our reads.

This will show box plots that represent distribution of quality for all bases at their respective positions in reads.

Install FastQC in the `dib_rotation` environment.
```
mamba install -y fastqc
```
Use the fastqc command on all reads in the `/raw_data` directory.
```
fastqc *.fastq*
```
This will create a `.zip` file and `.html` for each read. We can open the `.html` file in a browser to view the FastQC visualization. The `.zip` file can be viewed later.

### Transferring Data from FARM to your Computer
We can use `scp` to transfer files from the remote server to a local machine.

On a local terminal, use `scp`, the "secure copy" program, with the format `scp <FILE_TO_TRANSFER> <DESTINATION>`
```
scp -i ~/.ssh/farm rcantua@farm.cse.ucdavis.edu:~/2025_rotation/raw_data/*.html ./
```
>[!warning] Warning for Mac
If using ZSH, you must escape `\` out the asterisk.

Once the file is locally saved, you can open it in a browser.

### FastQC Output Graphs
:::info
#### Per base sequence quality
Shows the average quality score across all reads at all given positions.

#### Per sequence quality score
Shows the average quality score across all reads.

#### Per base sequence content
Shows the occurrence of any given base (adenine, thymine, cytosine, and guanine) across all reads at all given positions. (expect 25% for each; A=T and G=C)

#### Per sequence GC content
Shows the distribution of GC content across all reads.

#### Per base N content
Shows the average amount of N's (ambiguous nucleotide) at any position.
:::