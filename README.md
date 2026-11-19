# read_preprocessing
A nextflow workflow / pipeline to perform common NGS read preprocessing.  This can be used as stand-alone or can be used as subworkflow in another nextflow pipeline.

## Contents

- [Workflow steps](#Workflow-steps)
- [Running the pipeline](#Running-the-pipeline)
    - [Running from github](#Running-from-github)
    - [Running test datasets](#Running-test-datasets)
    - [Making sure that you are running the latest version](#Making-sure-that-you-are-running-the-latest-version)
    - [Running by cloning the pipeline's repository](#Running-by-cloning-the-pipeline-repository)
- [Inputs](#Inputs)
    - [Input fastq](#Input-fastq)
    - [Adapter sequences](#Adapter-sequences)
- [Outputs](#Outputs)
    - [Output directory](#Output-directory)
    - [Output file name prefixes ](#Output-file-name-prefixes)
    - [Optional deduplication](#Optional-deduplication)
- [Dependencies](#Dependencies)
    - [Nextflow](#Nextflow)
    - [Singularity ](#Singularity)


## Workflow steps

[This workflow](https://github.com/stenglein-lab/read_preprocessing/blob/main/subworkflows/stenglein-lab/preprocess_reads.nf) (or [subworkflow](https://www.nextflow.io/docs/latest/workflow.html#subworkflows)) performs common pre-processing steps on raw Illumina data.  The main steps include:

- Quality assessment of input reads with FASTQC.
- Trimming of adapter and low quality bases with cutadapt. 
- A second adapter trimming step using bbduk (in case any adapter sequences made it past cutadapt)
- Optional [removal](#Optional-deduplication) of duplicate reads or read pairs.
- Quality assessment of trimmed reads with FASTQC.
- Counting of reads at each step
- MultiQC report 

This takes advantage of nf-core [modules](https://nf-co.re/modules) for many of these components and the overall [nf-core](https://nf-co.re/) design philosophy.

## Running the pipeline

See the [dependencies section](#dependencies) below for information about the main dependencies required for running this pipeline(including nextflow and singularity).

### Running from github

The simplest way to run the pipeline is directly from github, like this:

```
nextflow run stenglein-lab/read_preprocessing -profile singularity -resume  --fastq_dir /path/to/fastq/directory 
```

You must specify the path to a directory containing input fastq (--fastq_dir).  See [this section](#inputs) for more information on required inputs.

### Running test datasets 

This repository includes several small [test datasets](./test/fastq) (fastq files) that can be used to validate pipeline function and to ensure that [dependencies](#dependencies) are satisfied.  To run these test datasets, run:

```
nextflow run stenglein-lab/read_preprocessing -profile test,singularity -resume 
```

Test output is placed in a `test/results` directory.

### Making sure that you are running the latest version

Nextflow will cache the pipeline in `$HOME/.nextflow/assets/` and continue to use the cached version, even if the pipeline has newer versions on github.  To remove the locally cached version, which will force nextflow to download and cache the latest version, run:

```
nextflow drop stenglein-lab/read_preprocessing
nextflow run stenglein-lab/read_preprocessing -profile singularity
```

Alternatively, you can just delete the cached pipeline directory:
```
rm -rf ~/.nextflow/assets/stenglein-lab/read_preprocessing/
```
Running nextflow pipelines from github is [described in more detail here](https://www.nextflow.io/docs/latest/sharing.html).

### Running by cloning the pipeline repository

It is also possible to download the pipeline code to a directory of your choosing.  This can be useful if, for instance, you want to modify or debug the code.  You can do this by cloning the repository (or a fork of the repository):

```
git clone https://github.com/stenglein-lab/read_preprocessing.git
cd read_preprocessing
nextflow run main.nf -resume --fastq_dir /path/to/fastq/directory -profile singularity
```

## Inputs

### Input fastq

Input sequence data is assumed to be Illumina single-end or paired-end data in separate read1 and read2 files (can be a mix of single and paired end).  Input fastq should  be gzip compressed.

The expected names of the fastq files are defined by the parameter `fastq_pattern`, whose default value is defined in nextflow.config as `*_R[12]_*.fastq*`.  This pattern can be overridden on the nextflow command line using the `--fastq_pattern` parameter.

The location of the fastq files is specified by the required `fastq_dir` parameter.  You can specify the location of multiple directories, with directory paths separated by commas (e.g., `--fastq_dir /fastq/dir1,/fastq/dir2`).

### Adapter sequences

Adapter sequences to be trimmed off of the 5′ and 3′ ends of reads by cutadapt are controlled by the parameters `--adapters_3p` and `--adapters_5p`.  These should be fasta files containing one or more sequences to be trimmed.  These default files are [here](https://github.com/stenglein-lab/read_preprocessing/blob/main/refseq/adapters_3p.fasta) and [here](https://github.com/stenglein-lab/read_preprocessing/blob/main/refseq/adapters_5p.fasta).  Adapter sequences for bbduk are controlled by the `--bbduk_adapters` parameter.  BBDuk default [here](https://github.com/stenglein-lab/read_preprocessing/blob/main/refseq/adapters_for_bbduk.fasta).


## Outputs

The main outputs of the pipeline are trimmed fastq files, which will be placed in a directory controlled by the `--fastq_out_dir` parameter.  By default, this directory will be `./results/trimmed_fastq`.  

### Optional deduplication

Duplicated reads or read pairs, presumably PCR duplicates, can be removed by setting the `--collapse_duplicate_reads` parameter, for instance:

```
nextflow run stenglein-lab/read_preprocessing -profile singularity -resume  --fastq_dir /path/to/fastq/directory 
```

In this case, fastq files containing trimmed but not deduplicated reads will be placed in the directory specified by `--fastq_out_dir`.  Trimmed and deduplicated fastq will be placed in a directory specified by `--fastq_out_dedup_dir`.


## Dependencies

This pipeline has two main dependencies: nextflow and singularity.  These programs must be installed on your computer to run this pipeline.

### Nextflow

To run the pipeline you will need to be working on a computer that has nextflow installed. Installation instructions can be [found here](https://www.nextflow.io/docs/latest/getstarted.html#installation).  To test if you have nextflow installed, run:

```
nextflow -version
```

This pipeline requires nextflow version > 22.10

### Singularity

The pipeline uses singularity containers to run programs like cutadapt, BLAST, and R.  To use these containers you must be running the pipeline on a computer that has [singularity](https://sylabs.io/singularity) [installed](https://sylabs.io/guides/latest/admin-guide/installation.html).  To test if singularity is installed, run:

```
singularity --version
```

There is no specified minimum version of singularity, but older versions of singularity (<~3.9) may not work.  the pipeline has been tested with singularity v3.9.5.

Singularity containers will be automatically downloaded and stored in a directory named `singularity_cacheDir` in your home directory.  They will only be downloaded once.
