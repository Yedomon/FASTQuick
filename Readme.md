[![Build Status](https://travis-ci.org/Griffan/FASTQuick.png?branch=master)](https://travis-ci.org/Griffan/FASTQuick)
[![GitHub Downloads](https://img.shields.io/github/downloads/Griffan/FASTQuick/total.svg?style=flat)](https://github.com/Griffan/FASTQuick/releases)

### OVERVIEW
   FASTQuick is an **ultra-fast** QC tool for NGS sequencing fastq files. It generates a comprehensive list of QC statistics, including **ancestry** and **contamination estimation**, at ~50x faster turnaround time than alignment-based QC tools.
   
### CONTENTS

- [INSTALL](#install)
- [GETTING STARTED](#getting-started)
- [FAQ](#faq)
- [WIKI PAGE](#wiki-page)
- [AUTHOR](#author)
- [COPYRIGHT](#copyright)

### INSTALL
To install FASTQuick, run the following series of commands.

```
git clone https://github.com/Griffan/FASTQuick.git
cd FASTQuick
mkdir build
cd build
cmake ..
make   
make test
```
If any test failed, refer to Testing/Temporary/LastTest.log for detailed error message(e.g. bcftools or samtools is missing).

Installation is complete if all tests finish successfully.

In case any required libraries is missing, you may specify customized installing path by replacing "cmake .." with:
```

For libhts:
  - cmake -DHTS_INCLUDE_DIRS=/hts_absolute_path/include/  -DHTS_LIBRARIES=/hts_absolute_path/lib/libhts.a ..

For bzip2:
  - cmake -DBZIP2_INCLUDE_DIRS=/bzip2_absolute_path/include/ -DBZIP2_LIBRARIES=/bzip2_absolute_path/lib/libbz2.a ..

For lzma:
  - cmake -DLZMA_INCLUDE_DIRS=/lzma_absolute_path/include/ -DLZMA_LIBRARIES=/lzma_absolute_path/lib/liblzma.a ..
```

A full list of required libraries and packages that are required to run the pipeline:
```
tools:
bwa
tabix
samtools
bcftools
pandoc
R

binary libraries:
libhts
zlib
libbzip2
libcurl
libssl

R libraries:
ggplot2
scales
knitr
rmarkdown
```

**Note** that if you use docker to deploy, the minimal memory requirement is 4GB.

### GETTING STARTED
Follow the procedures below to quickly get started using FASTQuick.

#### Clone and Install FASTQuick
First, to start using FASTQuick, clone and install the repository.
Please refer to [INSTALL](#install) for more comprehensive guide on how to download and install FASTQuick.

#### Perform a Test Run
To perform a test run to make sure that FASTQuick runs as expected with a very small-sized example (assuming that you are still inside `build` directory), run
```
cd ../example/
bash example.sh
```
for more example scripts to test whether the software tool works as expected or not, see [FAQ](#faq).

#### Download Reference Files
To run FASTQuick on real human sequence data, you need to download reference files using the following commands. (Before downloading, you may want to change your current directory.)
```
wget ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/phase2_reference_assembly_sequence/hs37d5.fa.gz
gzip -d hs37d5.fa.gz
bwa index hs37d5.fa
wget ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/dbsnp132_20101103.vcf.gz
gzip -d dbsnp132_20101103.vcf.gz
bgzip dbsnp132_20101103.vcf
tabix dbsnp132_20101103.vcf.gz
wget ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/release/20130502/supporting/accessible_genome_masks/20141020.strict_mask.whole_genome.bed
```
or from ncbi:
```
wget ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/technical/reference/phase2_reference_assembly_sequence/hs37d5.fa.gz
gzip -d hs37d5.fa.gz
bwa index hs37d5.fa
wget ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/technical/reference/dbsnp132_20101103.vcf.gz
gzip -d dbsnp132_20101103.vcf.gz
bgzip dbsnp132_20101103.vcf
tabix dbsnp132_20101103.vcf.gz
wget ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/release/20130502/supporting/accessible_genome_masks/20141020.strict_mask.whole_genome.bed
```
**Tips** if you are experiencing downloading difficulties, try to use the tool axel with cmdline "axel -n 10 ftp-url" to replace wget. 

**Note** that you do not need to run FASTQuick on GRCh38 reference because the alignment of FASTQuick will be used only for internal purpose and will provide very similar results regardless of which reference will be used.

#### Run FASTQuick for Your Own FASTQ Files
For simplicity, we prepared an all-in-one script to process the whole FASTQuick pipeline.

The pipeline can be generalized as "index ---> align ---> summarize ---> contamination && ancestry ---> visualization" (for detailed [workflow](https://github.com/Griffan/FASTQuick/wiki#general-workflow-of-fastquick)).

* index: FASTQuick will select eligible marker set, extract flanking sequences around markers, prepare SVD files based on reference genotype panel and build bwt indices.(**In this quick start, we have prepared predefined marker set and SVD files.**)
* align: Sequence reads are filtered and aligned to the reduced reference genome
* summarize: Summarize base level, read level, sample level statistics
* contamination && ancestry: Jointly estimate sample contamination level and genetic ancestry status
* visualization: Visualize the statistics collected above

Hence you may choose any start point of the pipeline (All | AllButIndex | Index | Align | Contamination | Ancestry | Visualize) for **--steps** in the command line.

```
${FASTQUICK_HOME}/bin/FASTQuick.sh \
--steps All \
--reference /path/to/hs37d5.fa \
--dbSNP /path/to/dbsnp132_20101103.vcf.gz \
--callableRegion /path/to/20141020.strict_mask.whole_genome.bed \
--candidateVCF ${FASTQUICK_HOME}/resource/1000g.phase3.10k.b37.vcf.gz \
--SVDPrefix ${FASTQUICK_HOME}/resource/1000g.phase3.10k.b37.vcf.gz \
--index <index.prefix> \
--output <sample.output.prefix> \
--fastq_1 <sample.input.R1.fastq.gz> \
--fastq_2 <sample.input.R2.fastq.gz>
```

* Users need to specify `<index.prefix>`, `<sample.output.prefix>`, `<sample.input.R1.fastq.gz>`, `<sample.input.R2.fastq.gz>`. 

* `${FASTQUICK_HOME}` represents the directory containing the FASTQuick repository. 

* Please replace `/path/to/` the directory that contains the downloaded reference files (or use `.` if everything happened in the same directory). You will need to specify the input and output file names denoted as `<...>`.

* **Note** that you only need to build indices once, hence `--steps AllButIndex` should be the preferred option once indices are ready.

#### Reference Files

From these reference files, we build index files which can be **shared** and **reused** by different samples:

**reference genome**(**--reference**) [hs37d5.fa](http://tinyurl.com/jvflzg3)

**dbSNP VCF**(**--dbSNP**) [dbsnp132_20101103.vcf.gz](http://tinyurl.com/sl2kgof)

**1000 strict masked region**(**--callableRegion**) [20141020.strict_mask.whole_genome.bed](http://tinyurl.com/sjhb5nn)

As for **candidate variant list**(**--candidateVCF**), we have prepared predefined marker set in ${FASTQUICK_HOME}/resource/ directory for the quick start, you can feed **--candidateVCF** with VCF file of the predefined markers, and **--SVDPrefix** with prefix of predefined SVD files. 

```
--candidateVCF ${FASTQUICK_HOME}/resource/10k.b37.FASTQuick.fa.SelectedSite.vcf
--SVDPrefix ${FASTQUICK_HOME}/resource/1000g.phase3.10k.b37.vcf.gz
```

Actually, these files are generated by FASTQuick.sh. You can always start from scratch by **not** specifying --SVDPrefix, which is time consuming for the indexing stage.(Refer to [FAQ-1](#faq-1))

**Note** that all these reference files are required to be the same build version with the reference genome.

#### Input Files
**--fastq_1** and **--fastq_2** expect pair-end fastq files(omit --fastq_2 for single-end dataset).
You can download fastq files of HG00553(PUR sample) from 1000 genome to reproduce the low-coverage [FinalReport.html](https://www.dropbox.com/s/7fbtpq82zduk4la/FinalReport.html?dl=1) in our example:

```
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR013/ERR013170/ERR013170_1.fastq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR013/ERR013170/ERR013170_2.fastq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR015/ERR015764/ERR015764_1.fastq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR015/ERR015764/ERR015764_2.fastq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR018/ERR018525/ERR018525_1.fastq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR018/ERR018525/ERR018525_2.fastq.gz
```
You can also use **--fastqList** to provide fastq files in format described in [FAQ-7](#faq-7)

#### Output Files
Once the process finished, you'll find summary statistics in various files starting with the same prefix(provided by **--output**). 
You also will find a similar [FinalReport.html](https://www.dropbox.com/s/7fbtpq82zduk4la/FinalReport.html?dl=1) in your output directory(base directory of prefix provided by **--output**). 

### FAQ

[1. How can I run FASTQuick with my own candidate variant list?](#faq-1)

[2. How can I run FASTQuick on target sequencing datasets?](#faq-2)

[3. What are the SVD files and how to generate them?](#faq-3)

[4. Why does index step failed (or take long time)?](#faq-4)

[5. What are the files in ${FASTQUICK_HOME}/resource/ directory?](#faq-5)

[6. Can I run FASTQuick step by step?](#faq-6)

[7. What is the format of fastq file list required by **--fastqList**?](#faq-7)

[8. How many markers are in use by default and how can I change the number of markers in use?](#faq-8)

[9. Can I run FASTQuick on GRCh38?](#faq-9)

<a href="#faq-1" id="faq-1"></a>

1. How can I run FASTQuick with my own candidate variant list?

    In this mode, FASTQuick will generate all the index files that are required. 
    Unlike cmdline example in [GETTING STARTED](#getting-started) where we prepared predefined marker set and SVD files for you, now you need to provide your own candidate variant list(shuffling recommended) for FASTQuick to choose from, e.g. Hapmap3 SNP set:
    ```
    wget -c ftp://gsapubftp-anonymous@ftp.broadinstitute.org/bundle/b37/hapmap_3.3.b37.vcf.gz
    bedtools shuffle hapmap_3.3.b37.vcf.gz > hapmap_3.3.b37.vcf.gz.shuffled.vcf
    ```
    you can use FASTQuick to select your own eligible marker set and build your own index files:
    The command line becomes:
    ```
    ${FASTQUICK_HOME}/bin/FASTQuick.sh \
    --steps All \
    --reference /path/to/hs37d5.fa \
    --dbSNP /path/to/dbsnp132_20101103.vcf.gz \
    --callableRegion /path/to/20141020.strict_mask.whole_genome.bed \
    --index <index.prefix> \
    --output <sample.output.prefix> \
    --fastq_1 <sample.input.R1.fastq.gz> \
    --fastq_2 <sample.input.R2.fastq.gz> \
    --candidateVCF /path/to/hapmap_3.3.b37.vcf.gz.shuffled.vcf \
    [--RefVCFList /path/to/ALL.chr*.phase3_shapeit2_mvncall_integrated_v5a.20130502.genotypes.vcf.gz.list]
    ```
    FASTQuick will also take longer time to prepare SVD files in this mode. As described in [FAQ-3]($faq-3), you can either let FASTQuick autimatically download [1000 genome variants VCF files](ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/release/20130502/) via internet or provide your own reference genotype vcf files with **--RefVCFList** (where each line hash a path of one vcf file).

    <a href="#faq-2" id="faq-2"></a>
2. How can I run FASTQuick on target sequencing datasets?

    In this mode, you will select markers only within your target regions. 
    Once the index files are generated, the usage of the pipeline can be the same as the whole genome examples [FAQ-1](#faq-1).
    Using cmdline example in  [GETTING STARTED](#getting-started) as a start point, you can enable **_target region_** mode by specifying **--targetRegion** with a bed format file.
    The command line becomes:
    ```
    ${FASTQUICK_HOME}/bin/FASTQuick.sh \
    --steps All \
    --reference /path/to/hs37d5.fa \
    --dbSNP /path/to/dbsnp132_20101103.vcf.gz \
    --callableRegion /path/to/20141020.strict_mask.whole_genome.bed \
    --index <index.prefix> \
    --output <sample.output.prefix> \
    --fastq_1 <sample.input.R1.fastq.gz> \
    --fastq_2 <sample.input.R2.fastq.gz> \
    --candidateVCF /path/to/hapmap_3.3.b37.vcf.gz.shuffled.vcf \
    --targetRegion <targetRegion.bed> 
    ```
    
    You can also speed the process up by specifying predefined SVD files even for target region mode:
    ```
    ${FASTQUICK_HOME}/bin/FASTQuick.sh \
    --steps All \
    --reference /path/to/hs37d5.fa \
    --dbSNP /path/to/dbsnp132_20101103.vcf.gz \
    --callableRegion /path/to/20141020.strict_mask.whole_genome.bed \
    --index <index.prefix> \
    --output <sample.output.prefix> \
    --fastq_1 <sample.input.R1.fastq.gz> \
    --fastq_2 <sample.input.R2.fastq.gz> \
    --candidateVCF /path/to/hapmap_3.3.b37.vcf.gz.shuffled.vcf \
    --targetRegion <targetRegion.bed> \
    --SVDPrefix ${FASTQUICK_HOME}/resource/exome/1000g.phase3.10k.b37.exome.vcf.gz \
    ```

    <a href="#faq-3" id="faq-3"></a>
    
3. What are these SVD files and how to generate them?

    You may have noticed that both in [GETTING STARTED](#getting-started) and in [Target Region Mode](<a href="#faq-2" id="faq-2"></a>) we specified predefined SVD files to speed the process up, for most cases you can directly use them too.    

    The SVD files are generated by performing SVD on on a panel of reference genotype(e.g. [1000 genome variants VCF files](ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/release/20130502/)).
        
    You can generate SVD files by specifying **--RefVCFList** with list of VCF files with genotype(e.g. [1000 genome variants VCF files](ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/release/20130502/)), and each line contains one VCF file.
    Or you can let FASTQuick.sh automatically download [1000 genome variants VCF files](ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/release/20130502/) and performing SVD for you, but this requires internet connection and will take longer time than using **--RefVCFList** option.

    <a href="#faq-4" id="faq-4"></a>

4. Why does index step failed (or take long time)?

    If you encounter this problem, please check the log files to ensure if the internet connection is successful throughout the running time. You may consider to use **--SVDPrefix** or **--RefVCFList** to skip certain procedures.

    <a href="#faq-5" id="faq-5"></a>

5. What are the files in ${FASTQUICK_HOME}/resource/ directory?
    
    This directory contains the predefined marker set and its corresponding SVD files. Because the SVD files(same prefix ending with .bed .mu .UD .V) usually can be reused, we prepare these files in advance to speed up the process.
    
    <a href="#faq-6" id="faq-6"></a>

6. Can I run FASTQuick step by step?

    We have prepared an example directory, in which you can find example scripts for each single step as template for customized usage.
    * the script **example.sh** is the simplified template for one-stop analysis.(our bin/FASTQuick.sh is more comprehensive and hence recommended)
    * the script **example.index.sh** is the template for selection new marker set and indexing reference data structures.
    * the script **example.align.sh** is the template for primary analysis.
    * the script **example.pop+con.sh** is the template to estimate contamination level and genetic ancestry of the intended sample.
    * the script **example.predefine.marker.index.sh** is the template to use pre-defined marker set to build indices.

    <a href="#faq-7" id="faq-7"></a>

7. What is the format of fastq file list required by **--fastqList**?

    FASTQuick.sh expects tab-delimited format as follows:
    ```
    read.group.A.read_1.fq.gz   read.group.A.read_2.fq.gz
    read.group.A.single.end.fq.gz
    read.group.B.read_1.fq.gz   read.group.B.read_2.fq.gz
    read.group.C.read_1.fq.gz   read.group.C.raed_2.fq.gz
    read.group.C.single.end.fq.gz
    ```
    
    <a href="#faq-8" id="faq-8"></a>
8. How many markers are in use by default and how can I change the number of markers in use?

    By default, we used 1000 long flanking region variants and 9000 short flanking region variants. You can modify this number by specifying **--ShortVariant** and **--LongVariant**.

    <a href="#faq-9" id="faq-9"></a>

9. Can I run FASTQuick on GRCh38?

    Yes, you can run FASTQuick on GRCh38. 
    However, FASTQuick does not reply on any standard/version of reference genome, because the alignment(coordinate system) will be used only for internal purpose and will provide very similar final report regardless of which version of reference will be used.
    We plan to provide GRCh38 based resource files later on, please stay tuned. 
  
### WIKI PAGE

We encourage users to refer to FASTQuick wiki page for more detailed description. [https://github.com/Griffan/FASTQuick/wiki]

### AUTHOR
Fan Zhang (email:fanzhang@umich.edu)

### COPYRIGHT
   The full FASTQuick package is distributed under MIT License.
