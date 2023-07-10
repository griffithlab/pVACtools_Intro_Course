
# Running pVACtools



## Learning Objectives

This chapter will cover:

- Starting an interactive Docker session
- Running pVACseq
- Running pVACfuse

## Starting Docker

In your Terminal execute the following command:


```bash
mkdir pVACtools_outputs

docker run \
-v ${PWD}/HCC1395_inputs:/HCC1395_inputs \
-v ${PWD}/pVACtools_outputs:/pVACtools_outputs \
-it griffithlab/pvactools:4.0.0 \
/bin/bash
```

This will pull the 4.0.0 version of the griffithlab/pvactools Docker image and
start an interactive session (`-it`) of that Docker image using the bash shell (`/bin/bash`). 
The `-v ${PWD}/HCC1395_inputs:/HCC1395_inputs`
part of the command will mount the
`HCC1395_inputs` folder at `/HCC1395_inputs` inside of the Docker container
so that you will have access to the input data from inside the Docker
container. The `-v ${PWD}/pVACtools_outputs:/pVACtools_outputs` part of the command
will mount the `pVACtools_outputs` folder you just created. We will write the
outputs from pVACseq and pVACfuse to that folder so that you will have access
to it once you exit the Docker image.

## Running pVACseq

pVACseq is used to identify neoantigens from missense, inframe indel, and
frameshift mutations. The pipeline uses a somatic VCF file as an input, which
represents variants identified in the tumor sample. The VEP annotations in the VCF file
provide the variant type of a variant and their consequence on individual gene transcripts
overlapping the genomic coordinates of the variant. The predicted amino acid change of
the variant for a particular transcript is used by pVACseq to calculate the mutated peptide sequence.

The pVACseq pipeline is run using the `pvacseq run` command.

### Required Parameters for pVACseq

The `pvacseq run` command takes a number of required parameters in the
following order:

- `vcf_file`: A VEP-annotated single- or multi-sample VCF containing genotype,
  transcript, Wildtype protein sequence, and Frameshift protein sequence
  information.
- `sample_name`: The name of the tumor sample being processed. When processing
  a multi-sample VCF the sample name must be a sample ID in the input VCF #CHROM
  header line. Only variants that are called (with a genotype/GT of 0/1 or 1/1) 
  in that sample will be processed.
- `allele(s)`: The name of the HLA allele(s) to use for epitope prediction. Multiple
  alleles can be specified using a comma-separated list. These should be the
  HLA alleles of your patient/sample. You might have clinical typing information for
  your patient. If not, you will need to computationally predict the patient's
  HLA type using software such as OptiType. The the HLA allele names should 
  be in the following format: `HLA-A*02:01`.  
- `prediction_algorithms`: The epitope prediction algorithms to use. Multiple
  prediction algorithms can be specified, separated by spaces. Use `all` to
  run all available prediction algorithms. pVACseq will automatically determine 
  which algorithms are valid for each HLA allele. 
- `output_dir`: The directory for writing all result files.

### Optional Parameters for pVACseq

The `pvacseq run` command offers quite a few optional arguments to fine-tune
your run. Here are a list of parameters we generally recommend:

- `--phased-proximal-variants-vcf`: This is an additional VCF file that
  includes both somatic and germline variants with phasing information. This
  file is used to identify variants near a somatic variant of interest and
  in-phase that would, as a result, change the predicted protein sequence
  around the somatic variant of interest and, thus, change the predicted
  neoantigens. Please note that pVACseq is currently only able to incorporate
  proximal missense variants so users should still manually investigate their
  candidates for other types of nearby variants (e.g. inframe and frameshift
  indels)
- `--normal-sample-name`: When using a tumor-normal input VCF, this parameter
  is used to identify the normal sample in the VCF in order to parse
  coverage metrics for the normal sample.
- `--iedb-install-directory`: For speed and reliability, we generally recommend
  that users use a standalone installation of the IEDB software. The pVACtools
  Docker containers already come with this software pre-installed in the
  `/opt/iedb` directory.
- `--allele-specific-binding-thresholds`: When filtering and tiering
  neoantigen candidates, one main criteria is the predicted peptide-MHC
  binding affinity. By default, pVACseq uses a cutoff of <500 nmol IC50.
  However, for some HLA alleles, other cutoffs are more appropriate depending
  on the distribution of binding affinities across peptides. Setting
  this flag enables allele-specific binding cutoffs as recommended by
  [IEDB](https://help.iedb.org/hc/en-us/articles/114094152371-What-thresholds-cut-offs-should-I-use-for-MHC-class-I-and-II-binding-predictions).
- `--allele-specific-anchors`: When considering a neoantigen candidate, only a
  subset of peptide positions are presented to the T cell receptor
  for recognition, while others are responsible for anchoring to the MHC, making
  these positional considerations critical for predicting T cell responses.
  Conventionally, the 1st, 2nd, n-1 and n position in a neoantigen candidate
  were considered anchors while recent studies [@Xia2023] have shown that
  these positions will depend on the HLA allele. Setting this flag will use
  allele-specific anchor locations where possible (we have predictions for ~300 common alleles).
- `--run-reference-proteome-similarity`: One consideration when selecting
  neoantigen candidates, is that the neoantigen should not occur natively in
  the patient's proteome. When this flag is set, pVACseq will search for each
  neoantigen candidate in the reference proteome and report any hits found.
  By default this is done using BLASTp but we recommend using a proteome FASTA
  file via the `--peptide-fasta` parameter to speed up this step. This will trigger
  a much faster k-mer based search strategy.
- `--pass-only`: By default, all variants that were called in the tumor sample
  are considered by pVACseq. This flag will lead pVACseq to skip variants that
  have a FILTER applied in the VCF to, e.g., exclude variants that were marked
  as low quality by the variant caller.
- `--percentile-threshold`: When considering the peptide-MHC binding affinity
  for filtering and prioritizing neoantigen candidates, by default only the
  IC50 value is being used. Setting this parameter will additionally also filter
  on the predicted percentile. We recommend a value of 0.01 (1%) for this
  threshold.

Additionally there are a number of parameters that might be useful depending
on your specific analysis needs:

- `--class-i-epitope-length` and `--class-ii-epitope-length`: By default 8,
  9, 10, 11 and 12, 13, 14, 15, 16, 17, 18 are set for these parameters,
  respectively, but different lengths might be desired.
- `--tumor-purity`: This parameter is used to bin variants into clonal and
  sub-clonal. This parameter might need to be adjusted based on the tumor
  purity of your data.
- `--problematic-amino-acids`: Some vaccine manufacturers will consider certain amino
  acids in the neoantigen candidates difficult to manufacture. For example, a
  Cysteine is commonly considered problematic as it makes the peptide
  unstable. This parameter allows users to set their own rules as to which
  peptides are considered problematic and peptides meeting those rules will be marked in the
  pVACseq results and deprioritized.
- `--threads`: This argument will allow pVACseq to run in multi-processing
  mode.
- `--keep-tmp-files`: Setting this flag will save intermediate files created by pVACseq.
- `--downstream-sequence-length`: For frameshift variants, the downstream
  sequence can potentially be very long, which can be computationally
  expensive. This parameter limits how many amino acids of the downstream
  sequence are included in the prediction. We often set a limit of `100`.

There are additional parameters in pVACseq that we won't discuss at this point
because the defaults are usually sufficient. To see all available parameters, you can
run `pvacseq run -h`.

### pVACseq Command

Given the considerations outlined above, let's run pVACseq on our sample data.

From the `optitype_normal_result.tsv` we know that the patient's class I alleles are
HLA-A\*29:02, HLA-B\*45:01, HLA-B\*82:02, and HLA-C\*06:02 (indicated that two of three class I
alleles are homozygous in this sample). We also have clinical typing information that confirms 
these class I alleles as well as identifying DQA1\*03:03, DQB1\*03:02, and DRB1\*04:05 as the 
patient's class II alleles.

Note that where needed pVACseq will automatically create HLA class II dimer combinations using
valid class II allele pairings.

To identify the tumor and normal sample names we will grep the VCF file for
the CHROM header:


```bash
zgrep CHROM /HCC1395_inputs/annotated.expression.vcf.gz
```

This shows that the tumor sample is named `HCC1395_TUMOR_DNA` and the normal sample is named `HCC1395_NORMAL_DNA`.

For our test run, please execute the `pvacseq run` command below. The
prediction run might take a while but pVACseq will output progress messages as
it runs through the pipeline.


```bash
pvacseq run \
/HCC1395_inputs/annotated.expression.vcf.gz \
HCC1395_TUMOR_DNA \
HLA-A*29:02,HLA-B*45:01,HLA-B*82:02,HLA-C*06:02,DQA1*03:03,DQB1*03:02,DRB1*04:05 \
all \
/pVACtools_outputs/pvacseq_predictions \
--normal-sample-name HCC1395_NORMAL_DNA \
--phased-proximal-variants-vcf /HCC1395_inputs/phased.vcf.gz \
--iedb-install-directory /opt/iedb \
--pass-only \
--allele-specific-binding-thresholds \
--percentile-threshold 0.01 \
--allele-specific-anchors \
--run-reference-proteome-similarity \
--peptide-fasta /HCC1395_inputs/Homo_sapiens.GRCh38.pep.all.fa.gz \
--problematic-amino-acids C \
--downstream-sequence-length 100 \
--n-threads 8 \
--keep-tmp-files
```

## Running pVACfuse

pVACfuse is run to in order to predict neoantigens from fusion events. The
pipeline uses annotated fusion calls from either AGFusion or Arriba for this
purpose. These annotators already include the fusion peptide sequence in their
outputs which pVACfuse uses to extract neoantigens around the fusion position.

The pVACfuse pipeline is run using the `pvacfuse run` command.

### Required Parameters for pVACfuse

The `pvacfuse run` command takes a number of required parameters in the
following order:

- `input_file`: An AGFusion output directory or Arriba fusion.tsv output file.
  For the purpose of this course, we will be running pVACfuse with AGFusion
  output.
- `sample_name`: The name of the tumor sample being processed.
- `allele(s)`: The name of the HLA allele to use for epitope prediction. Multiple
  alleles can be specified using a comma-separated list. These should be the
  HLA alleles of your patient. You might have clinical typing information for
  your patient. If not, you will need to computational predict the patient's
  HLA type using software such as OptiType.
- `prediction_algorithms`: The epitope prediction algorithms to use. Multiple
  prediction algorithms can be specified, separated by spaces. Use `all` to
  run all available prediction algorithms.
- `output_dir`: The directory for writing all result files.

### Optional Parameters for pVACfuse

In addition to the required parameters, the `pvacseq run` command also offers
optional arguments to fine-tune your run. You will find a lot of overlap
between pVACfuse and pVACseq parameters and the same general considerations
usually apply. Here are a list of parameters we generally recommend:

- `--starfusion-file`: Path to a `star-fusion.fusion_predictions.tsv` or
  `star-fusion.fusion_predictions.abridged.tsv`. This file is used to extract
  read support and expression information for each predicted fusion.
- `--iedb-install-directory`: For speed and reliability, we generally recommend
  that users use a standalone installation of the IEDB software. The pVACtools
  Docker containers already come with this software pre-installed in the
  `/opt/iedb` directory.
- `--allele-specific-binding-thresholds`: When filtering and tiering
  neoantigen candidates, one main criteria is the predicted peptide-MHC
  binding affinity. By default, pVACfuse uses a cutoff of <500 nmol IC50.
  However, for some HLA alleles, other cutoffs are more appropriate depending
  on the distribution of binding affinities across peptides. Setting
  this flag enables allele-specific binding cutoffs as recommended by
  [IEDB](https://help.iedb.org/hc/en-us/articles/114094152371-What-thresholds-cut-offs-should-I-use-for-MHC-class-I-and-II-binding-predictions).
- `--run-reference-proteome-similarity`: One consideration when selecting
  neoantigen candidates, is that the neoantigen should not occur natively in
  the patient's proteome. When this flag is set, pVACfuse will search for each
  neoantigen candidate in the reference proteome and report any hits found.
  By default this is done using BLASTp but we recommend using a proteome FASTA
  file via the `--peptide-fasta` parameter to speed up this step.
- `--percentile-threshold`: When considering the peptide-MHC binding affinity
  for filtering and prioritizing neoantigen candidates, by default only the
  IC50 value is being used. Setting this parameter will additionally also filter
  on the predicted percentile. We recommend a value of 0.01 (1%) for this
  threshold.

Additionally there are a number of parameters that might be useful depending
on your specific analysis needs:

- `--class-i-epitope-length` and `--class-ii-epitope-length`: By default 8,
  9, 10, 11 and 12, 13, 14, 15, 16, 17, 18 are set for these parameters,
  respectively, but different lengths might be desired.
- `--problematic-amino-acids`: Some vaccine manufacturers will consider certain amino
  acids in the neoantigen candidates difficult to manufacture. For example, a
  Cysteine is commonly considered problematic as it makes the peptide
  unstable. This parameter allows users to set their own rules as to which
  peptides are considered problematic and peptides meeting those rules will be marked in the
  pVACseq results and deprioritized.
- `--threads`: This argument will allow pVACfuse to run in multi-processing
  mode.
- `--keep-tmp-files`: Setting this flag will save intermediate files created by pVACfuse.
- `--downstream-sequence-length`: For frameshift fusions, the downstream
  sequence can potentially be very long, which can be computationally
  expensive. This parameter limits how many amino acids of the downstream
  sequence are included in the prediction. We often set a limit of `100`.

### pVACfuse Command

Given the considerations outlined above, let's run pVACfuse on our sample data.

As with pVACseq, we can use the `optitype_normal_result.tsv` file to identify the patient's
class I HLA alleles. These are HLA-A\*29:02, HLA-B\*45:01, HLA-B\*82:02, and HLA-C\*06:02.
We also have clinical typing information that confirms these class I alleles as well as 
identified DQA1\*03:03, DQB1\*03:02, and DRB1\*04:05 as the patient's class II alleles.

For pVACfuse the sample name is not used for any parsing so it doesn't need to
match any specific information in the AGFusion results. It is only used for
naming result files. For consistency we will use the same `HCC1395_TUMOR_DNA`
sample name we used in pVACfuse.

For our test run, please execute the `pvacfuse run` command below. The
prediction run might take a while but pVACfuse will output progress messages as
it runs through the pipeline.


```bash
pvacfuse run \
/HCC1395_inputs/agfusion_results \
HCC1395_TUMOR_DNA \
HLA-A*29:02,HLA-B*45:01,HLA-B*82:02,HLA-C*06:02,DQA1*03:03,DQB1*03:02,DRB1*04:05 \
all \
/pVACtools_outputs/pvacfuse_predictions \
--iedb-install-directory /opt/iedb \
--allele-specific-binding-thresholds \
--percentile-threshold 0.01 \
--run-reference-proteome-similarity \
--peptide-fasta /HCC1395_inputs/Homo_sapiens.GRCh38.pep.all.fa.gz \
--problematic-amino-acids C \
--downstream-sequence-length 100 \
--n-threads 8 \
--keep-tmp-files
```
