# :school: Práticas da Disciplina de Bioinformática (Bioinformatics Discipline Practices)

This is a repository for the Bioinformatics Discipline practices.

## :point_right: Setup the environment

#### How to install Conda?

* Download Anaconda software from  https://www.anaconda.com/products/individual
* Install it (https://docs.anaconda.com/anaconda/install)

#### How to create the environment using conda?

##### Option 01:

`Download the file [environment.yml](https://raw.githubusercontent.com/waldeyr/BioinfoDiscipline/main/environment.yml)`

`conda env create -f environment.yml`

##### Option 02:

`conda create -n DisciplinaBioinfo python=3.6 r=3.6 -y`

#### How to enter in the environment?

`conda activate DisciplinaBioinfo`

#### How to setup the channels (repositories) with the needed tools?

```
conda config --add channels bioconda
conda config --add channels conda-forge
```

#### How to install the needed tools into the DisciplinaBioinfo environment?

`conda install pandas numpy jupyterlab jupyter -y`

`conda install -c conda-forge readline=6.2 -y` 

`conda install -c bioconda sra-tools entrez-direct fastqc fastp spades quast star htseq seqtk samtools r-xml bioconductor-deseq2 -y`

* R packages (run it from the R prompt):

`install.packages('IRkernel')`

`install.packages("BiocManager")`

`BiocManager::install("DESeq2")`


## :notebook_with_decorative_cover: Practice 01 - De novo assembly of a Brazillian isolate of Sars-Cov-2 Genome

### Obtaining the raw material

* SARS-CoV-2 genome sequencing Rio Grande do Sul / Brazil, Dec 2020; Total RNA from SARS-CoV-2 positive samples was converted to cDNA. Viral whole-genome amplification was performed according to the Artic Network. Available in [SRR13510367](https://trace.ncbi.nlm.nih.gov/Traces/sra/?run=SRR13510367).

`fastq-dump --accession SRR13510367 --split-files --outdir rawdata -v`

### Generating a quality report for the sequenced reads

`fastqc rawdata/SRR13510367_1.fastq rawdata/SRR13510367_2.fastq`

### Filtering data with fastp

`mkdir filtered_data && fastp --thread 4 -p -q 30 -i rawdata/SRR13510367_1.fastq -I rawdata/SRR13510367_2.fastq -o filtered_data/SRR13510367_1_FILTERED.fastq -O filtered_data/SRR13510367_2_FILTERED.fastq && mv fastp.* filtered_data/`

### Running the asembly with Spades

`spades.py -t 4 -o assembly --careful -k 21,33,55 -1 filtered_data/SRR13510367_1_FILTERED.fastq -2 filtered_data/SRR13510367_2_FILTERED.fastq`

### generate a report about the assembly with Quast

* Download the Sars-Cov-2 reference genome

`esearch -db nucleotide -query "NC_045512.2" | efetch -format fasta > NC_045512.2.fasta`

* Performe the report

`quast.py assembly/scaffolds.fasta -r NC_045512.2.fasta`

## :notebook_with_decorative_cover: Practice 02 - [Transcriptome of human T cell stimulated with anti-CD3 antibody (Sousa, 2019)](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE112899)

Briefing: human peripheral blood mononuclear cells were purified from healthy volunteers blood and were cultured in the presence of the monoclonal antibody OKT3 or a recombinant fragment of humanized anti-CD3 (FvFcR) or recombinant fragment chimeric anti-CD3 (FvFcM).

* For the tutorial, we only will use the chromossome 22 and a monoclonal antibody OKT3 sample to make it feasible in a personal computer.

### Obtaining the raw material

* [Homo sapiens reference genome](http://www.ensembl.org/info/data/ftp/index.html)
* [Chromossome 22 DNA sequence](http://ftp.ensembl.org/pub/release-104/fasta/homo_sapiens/dna/Homo_sapiens.GRCh38.dna.chromosome.22.fa.gz)
* [Chromossome 22 GFF3 annotation](http://ftp.ensembl.org/pub/release-104/gff3/homo_sapiens/Homo_sapiens.GRCh38.104.chromosome.22.gff3.gz)
* [Chromossome 22 GTF annotation](http://ftp.ensembl.org/pub/release-104/gtf/homo_sapiens/Homo_sapiens.GRCh38.104.chr.gtf.gz)
* OKT3  replica 1 = SRR6974025
* FvFcR replica 1 = SRR6974027

`mkdir genome_reference && cd genome_reference`

`wget http://ftp.ensembl.org/pub/release-104/fasta/homo_sapiens/dna/Homo_sapiens.GRCh38.dna.chromosome.22.fa.gz`

`wget http://ftp.ensembl.org/pub/release-104/gff3/homo_sapiens/Homo_sapiens.GRCh38.104.chromosome.22.gff3.gz`

`gunzip Homo_sapiens.GRCh38.dna.chromosome.22.fa.gz && gunzip Homo_sapiens.GRCh38.104.chromosome.22.gff3.gz && cd ../`

`fastq-dump --accession SRR6974025 --outdir rawdata -v`

`fastq-dump --accession SRR6974027 --outdir rawdata -v`


### Filtering the reads quality using fastp

`fastqc rawdata/SRR6974025.fastq`

`fastqc rawdata/SRR6974027.fastq`

`mkdir filtered_data`

`fastp --thread 4 -p -q 30 -i rawdata/SRR6974025.fastq -o filtered_data/SRR6974025_FILTERED.fastq --verbose --average_qual 30`

`fastp --thread 4 -p -q 30 -i rawdata/SRR6974027.fastq -o filtered_data/SRR6974027_FILTERED.fastq --verbose --average_qual 30`

`mv fastp.* filtered_data/`

#### Sampling a reads subset (10 milion reads) to make it feasible in a notebook.

`seqtk sample -s100 filtered_data/SRR6974025_FILTERED.fastq 10000000 > filtered_data/SRR6974025_FILTERED_SUB.fastq`

`seqtk sample -s100 filtered_data/SRR6974027_FILTERED.fastq 10000000 > filtered_data/SRR6974027_FILTERED_SUB.fastq`


### Generating an index for the genome reference

`STAR --runThreadN 4 --runMode genomeGenerate --genomeDir genome_reference --genomeFastaFiles genome_reference/Homo_sapiens.GRCh38.dna.chromosome.22.fa --sjdbGTFfile genome_reference/Homo_sapiens.GRCh38.104.chromosome.22.gff3 --sjdbOverhang 99``

### Mapping the filtered reads to the genome using STAR

`STAR --genomeDir genome_reference --runThreadN 4 --readFilesIn filtered_data/SRR6974025_FILTERED_SUB.fastq --outFileNamePrefix SRR6974025 --outSAMtype BAM SortedByCoordinate --outSAMunmapped Within --outSAMattributes Standard --quantMode GeneCounts TranscriptomeSAM`

`STAR --genomeDir genome_reference --runThreadN 4 --readFilesIn filtered_data/SRR6974027_FILTERED_SUB.fastq --outFileNamePrefix SRR6974027 --outSAMtype BAM SortedByCoordinate --outSAMunmapped Within --outSAMattributes Standard --quantMode GeneCounts`

### Counting mapped genes with htseq-count

`htseq-count --nonunique all --format bam --stranded no --order pos --type exon --idattr gene_id SRR6974025Aligned.sortedByCoord.out.bam genome_reference/Homo_sapiens.GRCh38.104.chr.gtf > SRR6974025Aligned.counts
`

`htseq-count --nonunique all --format bam --stranded no --order pos --type exon --idattr gene_id SRR6974027Aligned.sortedByCoord.out.bam genome_reference/Homo_sapiens.GRCh38.104.chr.gtf > SRR6974027Aligned.counts`


## References
Sousa IG, Simi KCR, do Almo MM, Bezerra MAG et al. Gene expression profile of human T cells following a single stimulation of peripheral blood mononuclear cells with anti-CD3 antibodies. BMC Genomics 2019 Jul 19;20(1):593. PMID: 31324145
