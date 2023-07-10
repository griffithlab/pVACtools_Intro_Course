
# Prerequisites



## Learning Objectives

This chapter will cover the prerequisites for this course, including:

- Installing Docker
- Installing R Studio
- Downloading data files

## Docker

For the purpose of this course, we will be using Docker to run pVACseq and
pVACfuse.
Docker is a tool that is used to automate the deployment of applications
in lightweight containers so that applications can work efficiently in
different environments in isolation. We provide versioned Docker containers
for all pVACtools [releases](https://github.com/griffithlab/pVACtools/releases) 
via [Docker Hub using the griffithlab/pvactools image name](https://hub.docker.com/r/griffithlab/pvactools).

In order to use Docker, you will to download the [Docker Desktop software](https://www.docker.com/get-started/).
Please ensure you select the correct install package for your operating
system.

## Terminal

We will be running Docker from the command line on your preferred terminal
using the Docker command line interface (CLI). The Docker CLI is already
included with Docker Desktop. Most operating systems already
come with a Terminal application. If yours doesn't, you will need to first
install one.

## R Studio and R package dependencies

In order to use pVACview, you will need to download R. Please refer
[here](https://cran.rstudio.com/) for downloading R (version 3.5 and above
required). You may also take the additional step of [downloading R
studio](https://www.rstudio.com/products/rstudio/download/) if
you are not familiar with launching R Shiny from the command line.

Additionally, there are a number of packages you will need to install in your R/R studio:


```r
install.packages("shiny", dependencies=TRUE)
install.packages("ggplot2", dependencies=TRUE)
install.packages("DT", dependencies=TRUE)
install.packages("reshape2", dependencies=TRUE)
install.packages("jsonlite", dependencies=TRUE)
install.packages("tibble", dependencies=TRUE)
install.packages("tidyr", dependencies=TRUE)
install.packages("plyr", dependencies=TRUE)
install.packages("dplyr", dependencies=TRUE)
install.packages("shinydashboard", dependencies=TRUE)
install.packages("shinydashboardPlus", dependencies=TRUE)
install.packages("fresh", dependencies=TRUE)
install.packages("shinycssloaders", dependencies=TRUE)
install.packages("RCurl", dependencies=TRUE)
install.packages("curl", dependencies=TRUE)
install.packages("string", dependencies=TRUE)
install.packages("shinycssloaders", dependencies=TRUE)
```

## Data

For this course, we have put together a set of input data generated from the breast 
cancer cell line HCC1395 and a matched normal lymphoblastoid cell line HCC1395BL.
Data from this cell line is commonly used as test data in bioinformatics applications. 
For more information on these lines and the generation of test data, please refer to 
the data section of our precision medicine bioinformatics course: 
[here](https://pmbio.org/module-02-inputs/0002/05/01/Data/).

The input data consists of the following files:

For pVACseq:

- `annotated.expression.vcf.gz`: A somatic (tumor-normal) VCF and its tbi index file. The VCF has been
  annotated with VEP and has coverage and expression information added. It has also been annotated with 
  custom VEP plugins that provide wild type and mutant version of the full length protein sequences 
  predicted to arise from each transcript annotated with each variant.
- `phased.vcf.gz`: A phased tumor-germline VCF and its tbi index file to provide information about
  in-phase proximal variants that might alter the predicted peptide sequence around a somatic
  mutation of interest
- `optitype_normal_result.tsv`: A OptiType file with HLA allele typing predictions

For more detailed information on how the variant input file is created, please refer to the
[input file preparation](https://pvactools.readthedocs.io/en/latest/pvacseq/input_file_prep.html) 
section of the pVACtools docs

For pVACfuse:

- `agfusion_results`: A AGFusion output directory with annotated fusion calls
- `star-fusion.fusion_predictions.tsv`: A STARFusion prediction file with fusion read support
  and expression information

General:

- `Homo_sapiens.GRCh38.pep.all.fa.gz`: A reference proteome peptide FASTA to use
  for determining whether there are any reference matches of neoantigen candidates

To download this data, please run the following commands:


```bash
wget https://raw.githubusercontent.com/griffithlab/pVACtools_Intro_Course/main/HCC1395_inputs.zip
unzip HCC1395_inputs.zip
```

This course will not cover the required pre-processing steps for the pVACtools
input data but extensive instructions on how to prepare your own data for use
with pVACtools can be found at [pvactools.org](http://www.pvactools.org).

