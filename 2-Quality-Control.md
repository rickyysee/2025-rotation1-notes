# Quality Control
### Types of Quality Control
:::info
* **Adapter and Barcode Trimming**
Adapters are sequences of DNA that aid in the process of sequencing and are identical across a sample. Barcodes are unique sequences that serve to ID samples that are run in a single lane. Separating samples by barcode is called *demultiplexing*.
* **Quality Trimming**
You can "cut off" a read once the quality dips below a certain threshold.
* **K-mer Trimming**
K-mers are strings of DNA of length *k* in a given sequence. If there is a sequencing error, this will produce *k* erroneous k-mers.
:::
### When and How to Trim?
Trimming is a balance between removing incorrect nucleotides and keeping true ones. Below are some different cases.

* **Single-species genomic sequencing for assembly**
If you have sequenced *E. coli* isolate at 100X coverage, you would want to remove adapters and barcodes so they don't end up in the assembly. You would want to perform stringent quality and k-mer trimming. If you end up trimming 50% of your original reads, you would still be left with 50X coverage of high quality data.

* ***de novo* RNA-seq assembly**
If we sequences the transcriptome of a species without a reference transcriptome, we would have to be careful with k-mer trimming so we don't trim low abundance reads. RNA transcripts have different abundance profiles, which we would like to preserve. We would likely use light quality trimming (i.e. phred score of ~5) and trim only high-abundance k-mers.

* **Metagenome *de novo* assembly**
Since there are often low abundance organisms, we should be careful not to trim these out.

* **Metagenome read mapping**
When mapping reads to a reference, barcodes and adapters will often not affect mapping. However, it is safer to just remove them.

### Quality and Adapter Trimming with `fastp`
The FastQC report showed that Illumina Universal Adapter was present. The quality scores also drop towards the end of reads.
![image](https://hackmd.io/_uploads/HyMwHcrjhxg.png)
![image](https://hackmd.io/_uploads/Bk1ncrs3gg.png)

### Run `fastp`
Reminder that you should be in FARM on a compute node in the `dib_rotation` environment.
Install `fastp`.
```
mamba install -y fastp
```
Make a directory `/trim` to house the output of `fastp`.
```
cd ~/2025_rotation
mkdir -p trim
cd trim
```
Run `fastp` on the `SRR1976948` sample:
```
fastp --in1 ../raw_data/SRR1976948_1.fastq.gz \
  --in2 ../raw_data/SRR1976948_2.fastq.gz \
  --out1 SRR1976948_1.trim.fastq.gz \
  --out2 SRR1976948_2.trim.fastq.gz \
  --detect_adapter_for_pe \
  --qualified_quality_phred 4 \
  --length_required 31 --correction \
  --json SRR1976948.trim.json \
  --html SRR1976948.trim.html
```
>[!note] Command Arguments
>* `--in2`, `--in2` = the read1 and read2 input files
> * `--out1`, `--out2` = the read1 and read2 output files
> * `--detect_adapter_for_pe` = autodetect adapters for paired end (PE) reads and remove them
> * `--length_required` = removes reads shorter than a parameter (default is 15)
> * `--correction` = enables base correction if PE reads overlap
> * `--qualified_quality_phred` = threshold quality value. (default is >= 15)
> * `--html`, `--json` = file names for `fastp` reports

We set the Phred threshold to `4` to be lenient in our trimming since this is *de novo* metagenomics. A score of 10 means a base is 1/10 chance to be wrong. A 20 means a base is 1/100 chance to be wrong and so on.

Use `scp` to move the `fastp` html report to local computer.
```
scp -i ~/.ssh/farm rcantua@farm.cse.ucdavis.edu:~/2025_rotation/trim/\*.html ./
```
**Remember:** On Mac, you have to escape `\` the `*` wildcard.

### K-mer Trimming
After quality trimming with fastp, we still have errors, why?

1. The Phred score is purely a statistical measurement, so a base with a high score may still be wrong and one with a low score may be correct.
2. We used a low threshold to preserve coverage for downstream usage. (many techniques we will use do not suffer much from errors)

An alternative to quality trimming is trimming on k-mer abundance, or k-mer spectral error trimming. This table from [Zhang et al., 2014](https://journals.plos.org/plosone/article?id=10.1371%2Fjournal.pone.0101271) shows that k-mer trimming beats quality trimming in eliminating errors.
![image](https://hackmd.io/_uploads/B1NwmIshex.png)
> FP rate = false positive rate, "the frequency with which a given k-mer count will be incorrect when retrieved"

:::info
The logic is that if you have *low abundance* k-mers in a *high coverage* data set, they are most likely errors.

Since this is metagenomics data, we might have low coverage and high coverage data, so we don't simply want to get rid of low abundance k-mers.
:::
#### K-mer Trimming with `khmer`
The DIB lab's `khmer` project sorts reads into high and low abundance reads and only error trims the high abundance reads.
![image](https://hackmd.io/_uploads/Sk2XdUs3xx.png)

This will take 20GB of RAM and a few hours to complete. Let's start a new `srun` with the appropriate size for k-mer trimming.
```
srun --account=ctbrowngrp -p bmh -J khmer -t 20:00:00 --mem=21G -c 1 --pty bash
```
>[!note] Command Arguments
> * `--account=ctbrowngrp -p bmh` will specify you want the "big mem high" priority partition under Titus's group
> * adamgrp is a free tier 
> * `-c` specifies the number of CPUs per process

You will have to activate your environment again.
```
conda activate dib_rotation
```
Install `khmer`
```
mamba install -y khmer
```

Create a directory `/abundtrim` and `cd` into it. Now run a k-mer trim:

**The first line** of the following command will *interleave* our paired end reads, putting them in line in a single file where forward and reverse reads alternate on each line. **The second line** of the command performs k-mer trimming.
```
interleave-reads.py ../trim/SRR1976948_1.trim.fastq.gz ../trim/SRR1976948_2.trim.fastq.gz | trim-low-abund.py --gzip -C 3 -Z 18 -M 20e9 -V - -o SRR1976948.abundtrim.fq.gz
```
>[!note] FYI
> * The pipe `|` character sends the output of one command to the next.
> * We are referencing the trimmed files with a relative path `../trim/`. 

:::warning
**Issue**
Kept getting an `OSError: Generic StreamReadError`
![image](https://hackmd.io/_uploads/Hyf6l5j2xl.png)
From the output, it seems that the first script `interleave-reads.py` was failing due to lack of disk space, which caused the `trim-low-abund.py` script to fail as well.
:::
:::success
**Solution**
Went to `/raw_data` folder and deleted the raw sequence reads, which were each roughly 3 GB large. This freed up ~6GB of disk space and allowed the command to run.
>[!note]Reminder
Ask Titus how to get a slice of disk space from the shared pool.
:::
#### Assess Changes in K-mer Abundance
You can use `unique-kmers.py` script to see how many k-mers were removed with `khmer`.
```
unique-kmers.py ../trim/SRR1976948_1.trim.fastq.gz ../trim/SRR1976948_2.trim.fastq.gz
unique-kmers.py SRR1976948.abundtrim.fq.gz
```

Output from first command:
```
Estimated number of unique 32-mers in ../trim/SRR1976948_1.trim.fastq.gz: 271670733
Estimated number of unique 32-mers in ../trim/SRR1976948_2.trim.fastq.gz: 370138573
Total estimated number of unique 32-mers: 442248791
```
>[!warning]
These amounts of unique k-mers are slightly different than what Titus reported (`271670733` in read1, `370246495` in read2, and `442441435` total)

Output from second command:
```
Estimated number of unique 32-mers in SRR1976948.abundtrim.fq.gz: 329470996
Total estimated number of unique 32-mers: 329470996
```
>[!warning]
This amount is also different from Titus's (`329577970`)

The difference between the total unique k-mers in the quality trimmed files and those in the k-mer trimmed file is `112777795`. So, more than 112M k-mers were removed.