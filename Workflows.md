# Workflows, Automation, and Repeatability
## Scripts
Scripts are useful it automating large workflows into single files that hold the commands we want to run.
```
conda activate dib-rotation
cd ~/$(date +%Y)-$USER-rotation
nano run-qc.sh
```
In the script, put:
```
#! /bin/bash

cd ~/$(date +%Y)-$USER-rotation
mkdir -p quality
cd quality

ln -s ~/$(date +%Y)-$USER-rotation/raw-data/*.fastq.gz ./

printf "I see $(ls -1 *.fastq.gz | wc -l) files here.\n"

for infile in *_1.fastq.gz
  do
    name=$(basename ${infile} _1.fastq.gz)
    fastp --in1 ${name}_1.fastq.gz  --in2 ${name}_2.fastq.gz   --out1 ${name}_1.trim.fastq.gz --out2 $>
      --qualified_quality_phred 4  --length_required 31 --correction --json ${name}.trim.json --html $>
  done
```
We can make it executable by doing:
```
chmod +x ~/2020_rotation_project/run-qc.sh
```
If you run this script with `./run-qc.bash`, it will create a `quality/` directory, move into it, link the raw reads into it, then run `fastp` for the two read files.
## Snakemake
Snakemake is a workflow system. It is built on Python and meant to be user-friendly and scalable. It also works with Conda. Install it in the environment.
```
mamba install -y snakemake-minimal
```
`nano` a file called `Snakefile` (the default config file for Snakemake). Put into it:
```
SAMPLES = ["SRR1976948"]
rule all:
    input:
        expand("quality/{sample}_1.trim.fastq.gz", sample=SAMPLES),
        expand("quality/{sample}_2.trim.fastq.gz", sample=SAMPLES)

rule trim_reads:
    input:
        in1="raw-data/{sample}_1.fastq.gz",
        in2="raw-data/{sample}_2.fastq.gz",
    output:
        out1="quality/{sample}_1.trim.fastq.gz",
        out2="quality/{sample}_2.trim.fastq.gz",
        json="quality/{sample}.fastp.json",
        html="quality/{sample}.fastp.html"
    conda: "dib-rotation-environment.yaml"
    shell:
        """
        fastp --in1 {input.in1}  --in2 {input.in2}  \
        --out1 {output.out1} --out2 {output.out2}  \
        --detect_adapter_for_pe  --qualified_quality_phred 4 \
        --length_required 31 --correction \
        --json {output.json} --html {output.html}
        """
```
> * `snakemake -n` will do a dry run, just checking input files exist and that all files in `all` can be made
> * Might need to specify cores with `-c` when not doing a dry run
> * Can make Snakemake use specificed environment with `--use-conda`

Run Snakemake:
```
snakemake -c1 --use-conda
```
This will begin the DAG (directed acyclic graph).
## Bonus
Here, I will attempt to automate the entire rotation project with Snakemake.
```
cd ~/$(date +%Y)-$USER-rotation
mkdir -p auto
cd auto
```
### Environments
* `dib-rotation`
* `sgc`
* `kofamscan`
* `keggdecoder`

Make `.yaml` files for each.
```
conda env export -n dib-rotation -f environments/dib-rotation.yaml
conda env export -n sgc -f environments/sgc.yaml
conda env export -n kofamscan -f environments/kofamscan.yaml
conda env export -n keggdecoder -f environments/keggdecoder.yaml
```
### Snakefile
The DAG of Snakemake will automatically detect dependencies between rules to parallelize jobs.
:::spoiler First Snakemake
```
SAMPLES = ["SRR1976948"]
RUNS = [1,2]

rule all:
    input:
        fastqczip = expand("fastqc/{sample}_{run}_fastqc.zip", sample=SAMPLES, run=RUNS),
        fastqchtml = expand("fastqc/{sample}_{run}_fastqc.html", sample=SAMPLES, run=RUNS),
        khmer = expand("khmer/{sample}.abundtrim.fq.gz", sample=SAMPLES),
        gather = expand("sourmash/{sample}-gather.csv", sample=SAMPLES),

rule get_reads:
    output:
        out1 = "raw-data/{sample}_1.fastq.gz",
        out2 = "raw-data/{sample}_2.fastq.gz",
    shell:
        """
        mkdir -p raw-data
        wget -P raw-data/ ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR197/008/SRR1976948/SRR1976948_1.fastq.gz
        wget -P raw-data/ ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR197/008/SRR1976948/SRR1976948_2.fastq.gz
        """

rule fastqc:
    input:
        reads = expand("raw-data/{sample}_{run}.fastq.gz", sample=SAMPLES, run=RUNS),
    output:
        zip = expand("fastqc/{sample}_{run}_fastqc.zip", sample=SAMPLES, run=RUNS),
        html = expand("fastqc/{sample}_{run}_fastqc.html", sample=SAMPLES, run=RUNS),
    params:
        outdir="fastqc",
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        mkdir -p fastqc
        fastqc {input.reads} --outdir {params.outdir}
        """

rule trim_reads:
    input:
        in1 = "raw-data/{sample}_1.fastq.gz",
        in2 = "raw-data/{sample}_2.fastq.gz",
    output:
        out1 = "trim/{sample}_1.trim.fastq.gz",
        out2 = "trim/{sample}_2.trim.fastq.gz",
        json = "trim/{sample}.fastp.json",
        html = "trim/{sample}.fastp.html",
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        mkdir -p trim
        fastp --in1 {input.in1}  --in2 {input.in2}  \
        --out1 {output.out1} --out2 {output.out2}  \
        --detect_adapter_for_pe  --qualified_quality_phred 4 \
        --length_required 31 --correction \
        --json {output.json} --html {output.html}
        """

rule khmer:
    input:
        in1 = "trim/{sample}_1.trim.fastq.gz",
        in2 = "trim/{sample}_2.trim.fastq.gz",
    output:
        out = "khmer/{sample}.abundtrim.fq.gz",
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        mkdir -p khmer
        interleave-reads.py {input.in1} {input.in2} | trim-low-abund.py --gzip -C 3 -Z 18 -M 20e9 -V - -o {output}
        """

rule sourmash_sketch:
    input:
        in1 = "raw-data/{sample}_1.fastq.gz",
        in2 = "raw-data/{sample}_2.fastq.gz",
    output:
        sketch = "sourmash/{sample}.sig",
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        mkdir -p sourmash
        sourmash sketch dna -o {output} \
        --name {wildcards.sample} \
        -p scaled=2000,k=21,k=31,k=51,abund \
        {input}
        """

rule sourmash_database:
    output:
        "databases/genbank-k31.lca.json",
    shell:
        """
        mkdir -p databases
        curl -L https://osf.io/4f8n3/download -o databases/genbank-k31.lca.json.gz
        gunzip databases/genbank-k31.lca.json.gz
        """

rule sourmash_gather:
    input:
        sketch = "sourmash/{sample}.sig",
        database = "databases/genbank-k31.lca.json",
    output:
        gather = "sourmash/{sample}-gather.csv",
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        sourmash gather -o {output} \
        {input.sketch} {input.database}
        """
```
:::
Create a visual DAG with:
```
snakemake --dag | dot -Tpng > dag.png
```
Run Snakemake with:
```
snakemake -c2 --use-conda
```
:::warning
**NOTE**
When running this, it created 8 jobs (7 excluding the `all` rule). After a few hours, it was stuck at the terminal with some jobs complete and no error messages.

To troubleshoot, I checked the `.snakemake/log` folder and opened the most recent log file (`ls -lth` to see recent files). This showed me that 6 out of 8 jobs were complete and the last job needing to complete was the `khmer` rule. 

I think this is because I didn't allocate enough memory for `khmer`?
:::
```
# Original
# srun -A ctbrowngrp -p bmh -J snake -t 24:00:00 --mem=20gb -c 2 --pty bash

# New
srun -A ctbrowngrp -p bmh -J snake -t 48:00:00 --mem=30gb -c 2 --pty bash
```
I terminated Snakemake and will attempt to create a new `srun` session with sufficient memory and time. Will need to tell Snakemake to specifically run the incomplete process.
```
snakemake --rerun-incomplete -c2
```
Also, if you just need to run snakemake after a file has been modified or missing, use the `--rerun-triggers mtime`.
:::success
**Success!!**
![image](https://hackmd.io/_uploads/rksb9qG0gg.png)
:::
**NOTE**
Having trouble with `--use-conda` flag. Force channel setting?
:::spoiler Second Snakemake
```
SAMPLES = ["SRR1976948"]
RUNS = [1,2]

rule all:
    input:
        fastqczip = expand("fastqc/{sample}_{run}_fastqc.zip", sample=SAMPLES, run=RUNS),
        fastqchtml = expand("fastqc/{sample}_{run}_fastqc.html", sample=SAMPLES, run=RUNS),
        abund_gather = expand("sourmash/{sample}-abundtrim-gather.csv", sample=SAMPLES),
        raw_gather = expand("sourmash/{sample}-gather.csv", sample=SAMPLES),

rule get_reads:
    output:
        out1 = expand("raw-data/{sample}_1.fastq.gz", sample=SAMPLES),
        out2 = expand("raw-data/{sample}_2.fastq.gz", sample=SAMPLES),
    params:
        id = expand("{sample}", sample=SAMPLES),
    shell:
        """
        mkdir -p raw-data
        wget -P raw-data/ ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR197/008/{params.id}/{params.id}_1.fastq.gz
        wget -P raw-data/ ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR197/008/{params.id}/{params.id}_2.fastq.gz
        """

rule fastqc:
    input:
        reads = expand("raw-data/{sample}_{run}.fastq.gz", sample=SAMPLES, run=RUNS),
    output:
        zip = expand("fastqc/{sample}_{run}_fastqc.zip", sample=SAMPLES, run=RUNS),
        html = expand("fastqc/{sample}_{run}_fastqc.html", sample=SAMPLES, run=RUNS),
    params:
        outdir="fastqc",
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        mkdir -p fastqc
        fastqc {input.reads} --outdir {params.outdir}
        """

rule trim_reads:
    input:
        in1 = expand("raw-data/{sample}_1.fastq.gz", sample=SAMPLES),
        in2 = expand("raw-data/{sample}_2.fastq.gz", sample=SAMPLES),
    output:
        out1 = expand("trim/{sample}_1.trim.fastq.gz", sample=SAMPLES),
        out2 = expand("trim/{sample}_2.trim.fastq.gz", sample=SAMPLES),
        json = expand("trim/{sample}.fastp.json", sample=SAMPLES),
        html = expand("trim/{sample}.fastp.html", sample=SAMPLES),
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        mkdir -p trim
        fastp --in1 {input.in1}  --in2 {input.in2}  \
        --out1 {output.out1} --out2 {output.out2}  \
        --detect_adapter_for_pe  --qualified_quality_phred 4 \
        --length_required 31 --correction \
        --json {output.json} --html {output.html}
        """

rule khmer:
    input:
        in1 = expand("trim/{sample}_1.trim.fastq.gz", sample=SAMPLES),
        in2 = expand("trim/{sample}_2.trim.fastq.gz", sample=SAMPLES),
    output:
        out = expand("khmer/{sample}.abundtrim.fq.gz", sample=SAMPLES),
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        mkdir -p khmer
        interleave-reads.py {input.in1} {input.in2} | trim-low-abund.py --gzip -C 3 -Z 18 -M 20e9 -V - -o {output}
        """

rule raw_sketch:
    input:
        in1 = expand("raw-data/{sample}_1.fastq.gz", sample=SAMPLES),
        in2 = expand("raw-data/{sample}_2.fastq.gz", sample=SAMPLES),
    output:
        sketch = expand("sourmash/{sample}.sig", sample=SAMPLES),
    params:
        id = expand("{sample}", sample=SAMPLES),
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        mkdir -p sourmash
        sourmash sketch dna -o {output} \
        --name {params.id} \
        -p scaled=2000,k=21,k=31,k=51,abund \
        {input}
        """

rule get_database:
    output:
        "databases/genbank-k31.lca.json",
    shell:
        """
        mkdir -p databases
        curl -L https://osf.io/4f8n3/download -o databases/genbank-k31.lca.json.gz
        gunzip databases/genbank-k31.lca.json.gz
        """

rule raw_gather:
    input:
        sketch = expand("sourmash/{sample}.sig", sample=SAMPLES),
        database = expand("databases/genbank-k31.lca.json", sample=SAMPLES),
    output:
        gather = expand("sourmash/{sample}-gather.csv", sample=SAMPLES),
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        sourmash gather -o {output} \
        {input.sketch} {input.database}
        """

rule abundtrim_sketch:
    input:
        in1 = expand("khmer/{sample}.abundtrim.fq.gz", sample=SAMPLES),
    output:
        sketch = expand("sourmash/{sample}-abundtrim.sig", sample=SAMPLES),
    params:
        id = expand("{sample}", sample=SAMPLES),
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        mkdir -p sourmash
        sourmash sketch dna -o {output} \
        --name {params.id} \
        -p scaled=2000,k=21,k=31,k=51,abund \
        {input}
        """

rule abundtrim_gather:
    input:
        sketch = expand("sourmash/{sample}-abundtrim.sig", sample=SAMPLES),
        database = expand("databases/genbank-k31.lca.json", sample=SAMPLES),
    output:
        gather = expand("sourmash/{sample}-abundtrim-gather.csv", sample=SAMPLES),
    conda: "environments/dib-rotation.yaml"
    shell:
        """
        sourmash gather -o {output} \
        {input.sketch} {input.database}
        """
```        
:::
