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
- [Options](#Options)
    - [Adapter sequences](#Adapter-sequences)
    - [Always trim bases](#Always-trim-bases)
    - [Post trim minimum length](#Post-trim-minimum-length)
    - [BBDuk settings](#BBDUK-settings)
    - [Optional deduplication](#Optional-deduplication)
    - [Optional subsampling](#Optional-subsampling)
- [Outputs](#Outputs)
    - [Output directory](#Output-directory)
- [Dependencies](#Dependencies)
    - [Nextflow](#Nextflow)
    - [Singularity ](#Singularity)


## Workflow steps

[This workflow](https://github.com/stenglein-lab/read_preprocessing/blob/main/subworkflows/stenglein-lab/preprocess_reads.nf) (or [subworkflow](https://www.nextflow.io/docs/latest/workflow.html#subworkflows)) performs common pre-processing steps on raw Illumina data.  The main steps include:

- Quality assessment of input reads with FASTQC.
- Trimming of [adapter](#Adapter-sequences) and low quality bases with cutadapt. 
- A second trimming step using [BBDuk](#BBDUK-settings). 
- Optional [removal](#Optional-deduplication) of duplicate reads or read pairs.
- Optional [read subsampling](#Optional-subsampling)
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


## Options

### Adapter sequences

Adapter sequences to be trimmed off of the 5′ and 3′ ends of reads by cutadapt are controlled by the parameters `--adapters_3p` and `--adapters_5p`.  These should be fasta files containing one or more sequences to be trimmed.  These default files are [here](./refseq/adapters_3p.fasta) and [here](./refseq/adapters_5p.fasta).  Adapter sequences for bbduk are controlled by the `--bbduk_adapters` parameter.  The default BBDuk adapter sequences are [here](./refseq/adapters_for_bbduk.fasta).

### Always trim bases

The pipeline can always trim a certain number of bases off the beginning or ends of reads, controlled by the --always_trim_5p_bases (default: 0) or --always_trim_3p_bases (default 1) parameters.

### Post trim minimum length

Following trimming, the pipeline will retain reads that are longer than --post_trim_min_length (default: 30 bases). If you are working with very short reads (e.g. small RNA reads), you may wish to decrease this.

### BBDUK settings

The pipeline uses [BBDuk](https://bbmap.org/tools/bbduk) to perform additional trimming. Configurable BBDuk options include:

- `--bbduk_kmer_length` default: 15, used as input to BBDuk k parameter
- `--bbduk_hdist` default: 1, used as input to BBDuk hdist parameter
- `--bbduk_polyg_length` default: 8, used as option to BBDuk trimpolygright parameter
- `--bbduk_min_length` default: 30, used as input to BBDuk minlen parameter

### Optional deduplication

Duplicated reads or read pairs, presumably PCR duplicates, can be removed by setting the `--collapse_duplicate_reads` parameter, for instance:

```
nextflow run stenglein-lab/read_preprocessing -profile singularity -resume  --fastq_dir /path/to/fastq/directory --collapse_duplicate_reads
```

The workflow uses the [cd-hit-dup](https://github.com/weizhongli/cdhit/wiki/3.-User's-Guide#user-content-cdhitdup) tool to perform deduplication.

Configurable parameters for deduplication include:

- `--dedup_prefix_length` default: 30
- `--dedup_mismatches_allowed` default: 2
- `--params.dedup_percent_id` default: 0.99

Fastq files containing trimmed but not deduplicated reads will be placed in the `trimmed_fastq` [output](#outputs) subdirectory. Trimmed and deduplicated fastq will be placed in a `trimmed_deduplicated_fastq` subdirectory.

### Optional subsampling

The pipeline can optionally output a random subset of reads using [seqtk sample](https://github.com/lh3/seqtk)

To perform subsampling, specify the `--subsample_size` parameter (null by default).  Set this parameter to a fraction [0-1] to output a fraction of reads (e.g. 0.1: output 10% of reads) or to an integer >1 (e.g. 10000: output 10000 reads).

## Outputs

The main outputs of the pipeline are trimmed fastq files, which will be placed in a `trimmed_fastq` subdirectory of the main output directory, which is specified by the `--outdir` parameter (default: `results`).  

Other outputs go in various subdirectories of the main output directory.

Test outputs will go to `test/results`

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


### Citations

If you use this tool you should cite the tools it uses, including:

- [Nextflow](https://pubmed.ncbi.nlm.nih.gov/28398311/)
- [Singularity](https://doi.org/10.1371/journal.pone.0177459)
- [fastqc](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)
- [multiqc](https://doi.org/10.1093/bioinformatics/btw354)
- [cd-hit](https://pubmed.ncbi.nlm.nih.gov/23060610/)
- [cutadapt](https://doi.org/10.14806/ej.17.1.200)
- [BBDuk](https://bbmap.org/)
- [seqtk](https://github.com/lh3/seqtk)
