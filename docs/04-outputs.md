# Understanding pVACtools outputs



## Learning Objectives

This chapter will cover:

- Understanding the output files produced by pVACtools
- Interpreting the .filtered.tsv file
- Interpreting the .aggregated.tsv file

## pVACtools Output Files

pVACseq, pVACfuse, and pVACsplice all produce three main output files:

- The `all_epitopes.tsv` file is a TSV file with all predicted neoantigens and
  all information obtained during the run.
- The `filtered.tsv` file is the same structure as the all_epitopes.tsv file
  but the entries have been filtered down according to the thresholds set by
  the user during the run. The filters will be further explained in
  subsequent sections.
- The `aggregated.tsv` is a condensed output file that contains only the
  information most pertinent to interpret the results. It contains only
  the best neoantigen candidate for each variant. Our heuristic for
  determining the best neoantigen is described in subsequent sections of this
  course.

There are also a number of a secondary output files produced by pVACseq,
pVACfuse, and pVACsplice. The most important are:

- `aggregated.metrics.json`: The file is only produced by pVACseq. It contains
  metadata needed for visualizing your results in pVACview.
- `aggregated.tsv.reference_matches`: This file is created when the
  reference proteome match feature is enabled during a run. It contains
  detailed information about the reference matches found, if there are any.

## Interpreting the filtered.tsv File

The filtered.tsv file takes all the predicted neoantigens from the
all_epitopes.tsv file and applies a number of filters to it. Filters are
applied consecutively, meaning that only the entries passing the first filter
will be passed along to the second filter, and so on. Only neoantigens
passing all filters will be reported in this file.

### Binding Filter

The binding filter's primary function is to filter neoantigen candidates on
their IC50 binding affinity to an HLA allele. Because pVACtools allows users
to run more than one prediction algorithm, we then apply two summarization
methods on the calls for each neoantigen candidate and HLA allele combination:
(1) pVACtools calculates the median IC50 binding affinity for all selected prediction
algorithms (reported in the `Median [MT] IC50 Score` column), and (2) pVACtools selects
the IC50 binding affinity prediction with the lowest value (reported in the
`Best [MT] IC50 Score` column). By default,
the binding filter is applied to the median IC50 score unless
users set the `--top-score-metric` parameter to `lowest`.

The binding filter discards candidates where the binding affinity is above the
`--binding-threshold` (default: 500). However, users may set the
`--allele-specific-binding-thresholds` flag in order to use differing binding
thresholds depending on the HLA allele of the prediction, as recommended by
[IEDB](https://help.iedb.org/hc/en-us/articles/114094152371-What-thresholds-cut-offs-should-I-use-for-MHC-class-I-and-II-binding-predictions).
Custom thresholds are available for the most common 76 class I HLA alleles.
For all others, the `--binding-threshold` value is used.

In addition to the binding affinity, other optional parameters can be set to
enabled additional filtering on related metrics:

- `--minimum-fold-change`: The fold change is the ratio of the mutant binding affinity to
  the wild-type binding affinity, also called agretopicity. A fold change of 1
  means that the mutant is a better binder than the wild type. pVACtools
  calculates this ratio for both the median as well as the lowest values.
  Which one is filtered on for this metric depends again on the
  `--top-score-metric` set. When a minimum fold change parameter is set, the binding filter
  discards any prediction with a agretopicity below the set cutoff. This
  parameter is not available in pVACfuse because there is no matched wildtype
  peptide for each neoantigen candidate.
- `--percentile-threshold`: The prediction algorithms supported by pVACtools
  also report a percentile score that represents where each neoantigen's predicted
  affinity falls in the range of other values for an HLA allele. Similar to
  the binding affinity itself, pVACtools report the median and the lowest
  percentile scores for the range of scores reported by the prediction
  algorithms chosen by the user and which on is used for filtering is again
  controlled by the `--top-score-metric` parameter.
- `--percentile-threshold-strategy`: When running pVACtools with a
  `--percentile-threshold` set, this parameter will influence how both the
  binding cutoff and the percentile cutoff are applied. The default,
  `conservative`, will require a candidate to pass both the binding and the
  percentile threshold while the `exploratory` option will require a candiate
  to only pass either the binding or the percentile threshold.

### Coverage Filter

The Coverage Filter is generally used to filter out variants that don't have
enough read support or expression. This ensures that the remaining variants
are not just artifacts and that the genes are actually expressed in the
patient's RNA.

For both pVACseq and pVACsplice, this generally relies on your VCF being annotated with coverage
and expression data. In our example, the VCF has already been annotated with
this data. For more information about how to add [coverage](https://pvactools.readthedocs.io/en/latest/pvacseq/input_file_prep/readcounts.html)
and [expression data](https://pvactools.readthedocs.io/en/latest/pvacseq/input_file_prep/expression.html)
to your own VCFs, please see our docs.
Additionally, filtering on the normal DNA depth and variant allele frequency
(VAF) requires your VCF to be a tumor-normal sample VCF and the normal sample
to be identified in your pVACseq/pVACsplice run using the `--normal-sample-name`
parameter. If a coverage metric doesn't apply because the underlying data is
not available, `NA` is reported by pVACtools. By default, the filter will skip
evaluating a coverage criteria when a neoantigen's value for it is `NA`.

The following thresholds are applied in pVACseq and pVACsplice by this filter:

- `--normal-cov`: Normal coverage cutoff. Minimum number of required reads in the normal DNA (default: 5).
- `--tdna-cov`: Tumor DNA coverage cutoff. Minimum number of required reads in the tumor DNA (default: 10).
- `--trna-cov`: Tumor RNA coverage cutoff. Minimum number of required reads in the tumor RNA (default: 10).
- `--normal-vaf`: Normal VAF cutoff. Only sites BELOW this cutoff in the normal DNA will be considered (default: 0.02).
- `--tdna-vaf`: Tumor DNA VAF cutoff. Only sites above this cutoff will be considered (default: 0.25).
- `--trna-vaf`: Tumor RNA VAF cutoff. Only sites above this cutoff will be considered (default: 0.25).
- `--expn-val`: Gene and Transcript expression cutoff. Only sites above this cutoff will be considered (default: 1.0).

For pVACfuse, this filter evaluates a fusion variant's fusion read support and fusion transcript expression.
Arriba natively outputs a number of read metrics. These are the number of supporting split fragments with an anchor in
gene1 or gene2, respectively, as well as the number of pairs (fragments) of discordant mates supporting the fusion
(a.k.a. spanning reads or bridge reads). The sum of these three values is
reported as Read Support in pVACfuse. For AGFusion, the read support is parsed
from the `--starfusion-file` as the sum of the JunctionReadCount and
SpanningFragCount. For both types of input, the fusion transcript expression is
parsed from the `--starfusion-file`, when provided. This is reported as FFPM
(fusion fragments per million total reads).

The following thresholds are applied in pVACfuse by this filter:

- `--read-support`: Read Support cutoff. Sites above this cutoff will be considered (default: 5).
- `--expn-val`: Expression cutoff. Sites above this cutoff will be considered (default: 0.1).

### Transcript Filter

The Transcript Filter removes neoantigens resulting from transcripts that are
considered poor candidates. To determine whether a transcript is poor, the
`--transcript-prioritization-strategy` parameter is used. This parameter
defines a list of criteria to consider. These are:

- `mane_select`: MANE Select status of the transcript
- `canonical`: Canonical status of the transcripts
- `tsl`: Whether or not the Transcript Support Level (TSL) meets the
  `--maximum-transcript-support-level`, as defined [by Ensembl](https://grch37.ensembl.org/info/genome/genebuild/transcript_quality_tags.html#tsl).

Users may specify one or more of these criteria as their strategy. A transcript
meeting at least one of the specified criteria is considered a
good transcript and passes the transcript filter. Only transcripts meeting none of the
specified criteria will be filtered out.

This filter is currently only run by pVACseq and pVACsplice.

### Top Score Filter

The Top Score Filter will attempt to determine the best neoantigen candidate
for each variants.

For pVACseq it works as follows. Given a set of neoantigen candidates for a
variant we first group the transcripts into sets where all transcripts in a set
code for the same set of neoantigen candidates. For each transcript set we then
determine the best neoantigen candidate as follows:

- Pick all neoantigens with a variant transcript that doesn't have a Transcript CDS Flag
- Of the remaining candidates, pick the ones with a variant transcript that have a protein_coding Biotype
- Of the remaining candidates, pick the ones with a passing transcript according
  to the selected `--transcript-prioritization-strategy` and `--maximum-transcript-support-level`.
- Of the remaining candidates, pick the entries with no Problematic Positions.
- Of the remaining candidates, pick the ones passing the Anchor Criteria (explained in
  more detail further below).
- Sort the remaining candidates on the lowest MT IC50 Score or Percentile (Median or Best
  depending on the `--top-score-metric`, Score or Percentile depending on the
  `--top-score-metric2`), the transcript's MANE Select Status, the
  transcript's Canonical status, the transcript's TSL, the transcript length,
  and the transcript expression. Select the highest sorted entry.

This filter then reports the best neoantigen candidate for each transcript set.

For pVACfuse, the neoantigen candidate for each fusion are similarly grouped
into sets where all transcript1-transcript2 combinations in a set code for the
same set of neoantigen candidates. From there, the best neoantigen candidate
for each transcript set is determined by picking the candidate with the lowest
MT IC50 Score (Median or Best depending on the `--top-score-metric`) and the
highest fusion transcript expression.

For pVACsplice, the neoantigen candidates are grouped into sets with the same splice
site Junction. From there, the best neoantigen candidate for each set is
determined very similarly to pVACseq:

- Pick all neoantigens with a variant transcript that doesn't have a Transcript CDS Flag
- Of the remaining candidates, pick the ones with a variant transcript that have a protein_coding Biotype
- Of the remaining candidates, pick the ones with a passing transcript according
  to the selected `--transcript-prioritization-strategy` and `--maximum-transcript-support-level`.
- Of the remaining candidates, pick the entries with no Problematic Positions.
- Of the remaining candidates, pick the ones passing the Anchor Criteria (explained in
  more detail further below).
- Sort the remaining candidates on the lowest MT IC50 Score or Percentile (Median or Best
  depending on the `--top-score-metric`, Score or Percentile depending on the
  `--top-score-metric2`), the transcript's MANE Select Status, the
  transcript's Canonical status, the transcript's TSL, the transcript WT
  Protein Length, and the transcript expression. Select the highest sorted entry.

## Interpreting the aggregated.tsv File

The `aggregated.tsv` is a condensed output file that shows the best neoantigen
candidate for each variant and reports only the information most pertinent to
interpreting the results. It also assigns each of the selected neoantigen candidates
a tier based on its suitability for vaccine manufacturing.

Only epitopes meeting the `--aggregate-inclusion-threshold` are included in this report
(default: 5000). Depending on the value used for the `--top-score-metric`, all neoantigen
candidates with a Median or Best MT IC50 Score below the selected `--aggregate-inclusion-threshold`
are included in creating this report. If this cutoff yields a lot of candidates, the
included neoantigens are pared to the best `--aggregate-inclusion-count-limit` candidates.

### Determining the Best Transcript and Best Peptide of a Variant

In pVACseq, for each variant, all neoantigen candidates meeting the `--aggregate-inclusion-threshold` are evaluated as follows:

- Pick all neoantigens with a variant transcript that doesn't have a Transcript CDS Flag
- Of the remaining candidates, pick the ones with a variant transcript that have a protein_coding Biotype
- Of the remaining candidates, pick the ones with a passing transcript according
  to the selected `--transcript-prioritization-strategy` and `--maximum-transcript-support-level`.
- Of the remaining candidates, pick the entries with no Problematic Positions.
- Of the remaining candidates, pick the ones passing the Anchor Criteria (explained in
  more detail further below).
- Sort the remaining candidates on the lowest MT IC50 Score or Percentile (Median or Best
  depending on the `--top-score-metric`, Score or Percentile depending on the
  `--top-score-metric2`), the transcript's MANE Select Status, the
  transcript's Canonical status, the transcript's TSL, the transcript length,
  and the transcript expression. Select the highest sorted entry.

In pVACfuse, the neoantigen candidate with the lowest IC50 binding affinity for each variant
and highest transcript expression is selected.
The value used for the `--top-score-metric` determines whether the lowest or
median binding affinity is used for this comparison.

For pVACsplice, the best neoantigen candidate for each variant is determined very similarly to pVACseq:

- Pick all neoantigens with a variant transcript that doesn't have a Transcript CDS Flag
- Of the remaining candidates, pick the ones with a variant transcript that have a protein_coding Biotype
- Of the remaining candidates, pick the ones with a passing transcript according
  to the selected `--transcript-prioritization-strategy` and `--maximum-transcript-support-level`.
- Of the remaining candidates, pick the entries with no Problematic Positions.
- Of the remaining candidates, pick the ones passing the Anchor Criteria (explained in
  more detail further below).
- Sort the remaining candidates on the lowest MT IC50 Score or Percentile (Median or Best
  depending on the `--top-score-metric`, Score or Percentile depending on the
  `--top-score-metric2`), the transcript's MANE Select Status, the
  transcript's Canonical status, the transcript's TSL, the transcript WT
  Protein Length, and the transcript expression. Select the highest sorted entry.

The chosen entry determines the best neoantigen candidate and the best
transcript coding for it.

### Tier and Tiering Criteria

For the purpose of assigning tiers, each best peptide is evaluated by a set of
criteria. These criteria and the available tiers differ from tool to tool.

#### Tiering in pVACseq

The Tiers available in pVACseq are:


| Tier | Criteria |
|------|----------|
| Pass | Best Peptide passes the binding, reference match, expression, transcript, clonal, problematic position, and anchor criteria |
| PoorBinder | Best Peptide fails the binding criteria but passes the reference match, expression, transcript, clonal, problematic position, and anchor criteria |
| RefMatch | Best Peptide fails the reference match criteria but passes the binding, expression, transcript, clonal, problematic position, and anchor criteria |
| PoorTranscript | Best Peptide fails the transcript criteria but passes the binding, reference match, expression, clonal, problematic position, and anchor criteria |
| LowExpr | Best Peptide meets the low expression criteria and passes the binding, reference match, transcript, clonal, problematic position, and anchor criteria |
| Anchor | Best Peptide fails the anchor criteria but  passes the binding, reference match, expression, transcript, clonal, and problematic position criteria |
| Subclonal | Best Peptide fails the clonal criteria but passes the binding, reference match, expression, transcript, problematic position, and anchor criteria |
| ProbPos | Best Peptide fails the problematic position criteria but passes the binding, reference match, expression, transcript, clonal, and anchor criteria |
| Poor | Best Peptide doesn’t fit in any of the above tiers, usually if it fails two or more criteria |
| NoExpr | Best Peptide is not expressed (RNA Expr == 0 or RNA VAF == 0) |


**Criteria Details**


| Criteria | Description | Evaluation |
|----------|-------------|------------|
| Binding Criteria | Pass if Best Peptide is a strong binder | binding score criteria: `IC50 MT < --binding-threshold` (`--allele-specific-binding-thresholds` flag is respected)<br/>percentile score criteria (if `--percentile-threshold` parameters is set): `%ile MT < --percentile-threshold` (if parameter is set)<br/>`conservative` `--percentile-threshold-strategy`: needs to pass BOTh the binding score criteria AND the percentile score criteria<br/>`exploratory` `--percentile-threshold-strategy`: needs to pass EITHER the binding score criteria OR the percentile score criteria|
| Expression Criteria | Pass if Best Transcript is expressed | Allele Expr > `--trna-vaf` * `--expn-val` |
| Reference Match Criteria | Pass if there are no reference protome matches | `Ref Match == False` |
| Transcript Criteria | Pass if Best Transcript matches any of the user-specified `--transcript-prioritization-strategy` criteria | `TSL <= --maximum-transcript-support level` (if strategy includes `tsl`)<br/>`MANE Select == True` (if strategy includes `mane_select`)<br/>`Canonical == True` (if strategy includes `canonical`) |
| Low Expression Criteria | Peptide has low expression or no expression but RNA VAF and coverage | (0 < Allele Expr < `--trna-vaf` * `--expn-val`) OR (RNA Expr == 0 AND RNA Depth > `--trna-cov` AND RNA VAF > `--trna-vaf`) |
| Anchor Criteria | Fail if all mutated amino acids of the Best Peptide (Pos) are at an anchor position and the WT peptide has good binding (IC50 WT < `--binding-threshold`). `--allele-specific-binding-thresholds` flag is respected. |
| Clonal Criteria | Best Peptide is likely in the founding clone of the tumor | DNA VAF > `--tumor-purity` / 4 |
| Problematic Position Criteria | Best Peptide does not contain a problematic amino acid as defined by the `--problematic-amino-acids` parameter | `Prob Pos == None`


#### Tiering in pVACfuse

The Tiers available in pVACfuse are:


| Tier | Criteria |
|------|---------|
| Pass | Best Peptide passes the binding, reference match, read support, expression, and problematic position criteria |
| PoorBinder | Best Peptide fails the binding criteria but passes the reference match, read support, expression, and problematic position criteria |
| RefMatch | Best Peptide fails the reference match criteria but passes the binding, read support, expression, and problematic position criteria |
| LowReadSupport | Best Peptide fails the read support criteria but passes the binding, reference match, expression, and problematic position criteria |
| LowExpr | Best Peptide fails the expression criteria but passes the binding, reference match, read support, and problematic position criteria |
| ProbPos | Best Peptide fails the problematic position criteria but passes the binding, reference match, read support, and expression |
| Poor | Best Peptide doesn’t fit any of the above tiers, usually if it fails two or more criteria |


**Criteria Details**


| Criteria | Description | Evaluation |
|----------|-------------|------------|
| Binding Criteria | Pass if Best Peptide is a strong binder | binding score criteria: `IC50 MT < --binding-threshold` (`--allele-specific-binding-thresholds` flag is respected)<br/>percentile score criteria (if `--percentile-threshold` parameters is set): `%ile MT < --percentile-threshold` (if parameter is set)<br/>`conservative` `--percentile-threshold-strategy`: needs to pass BOTh the binding score criteria AND the percentile score criteria<br/>`exploratory` `--percentile-threshold-strategy`: needs to pass EITHER the binding score criteria OR the percentile score criteria|
| Expression Criteria | Pass if variant has read support | Read Support < `--read-support` |
| Reference Match Criteria | Pass if there are no reference protome matches | `Ref Match == False` |
| Problematic Position Criteria | Best Peptide does not contain a problematic amino acid as defined by the `--problematic-amino-acids` parameter | `Prob Pos == None`


#### Tiering in pVACsplice

The Tiers available in pVACsplice are:


| Tier | Criteria |
|------|----------|
| Pass | Best Peptide passes the binding, reference match, expression, transcript, clonal, and problematic position criteria |
| PoorBinder | Best Peptide fails the binding criteria but passes the reference match, expression, transcript, clonal, and problematic position criteria |
| RefMatch | Best Peptide fails the reference match criteria but passes the binding, expression, transcript, clonal, and problematic position criteria
| PoorTranscript | Best Peptide fails the transcript criteria but passes the binding, reference match, expression, clonal, and problematic position criteria |
| LowExpr | Best Peptide meets the low expression criteria and passes the binding, reference match, transcript, clonal, and problematic position criteria |
| Subclonal | Best Peptide fails the clonal criteria but passes the binding, reference match, expression, transcript, and problematic position criteria |
| ProbPos | Best Peptide fails the problematic position criteria but passes the binding, reference match, expression, transcript, and clonal criteria |
| Poor | Best Peptide doesn’t fit in any of the above tiers, usually if it fails two or more criteria |
| NoExpr | Best Peptide is not expressed (RNA Expr == 0 or RNA VAF == 0) |


**Criteria Details**


| Criteria | Description | Evaluation |
|----------|-------------|------------|
| Binding Criteria | Pass if Best Peptide is a strong binder | binding score criteria: `IC50 MT < --binding-threshold` (`--allele-specific-binding-thresholds` flag is respected)<br/>percentile score criteria (if `--percentile-threshold` parameters is set): `%ile MT < --percentile-threshold` (if parameter is set)<br/>`conservative` `--percentile-threshold-strategy`: needs to pass BOTh the binding score criteria AND the percentile score criteria<br/>`exploratory` `--percentile-threshold-strategy`: needs to pass EITHER the binding score criteria OR the percentile score criteria|
| Expression Criteria | Pass if Best Transcript is expressed | Allele Expr > `--trna-vaf` * `--expn-val` |
| Reference Match Criteria | Pass if there are no reference protome matches | `Ref Match == False` |
| Transcript Criteria | Pass if Best Transcript matches any of the user-specified `--transcript-prioritization-strategy` criteria | `TSL <= --maximum-transcript-support level` (if strategy includes `tsl`)<br/>`MANE Select == True` (if strategy includes `mane_select`)<br/>`Canonical == True` (if strategy includes `canonical`) |
| Low Expression Criteria | Peptide has low expression or no expression but RNA VAF and coverage | (0 < Allele Expr < `--trna-vaf` * `--expn-val`) OR (RNA Expr == 0 AND RNA Depth > `--trna-cov` AND RNA VAF > `--trna-vaf`) |
| Clonal Criteria | Best Peptide is likely in the founding clone of the tumor | DNA VAF > `--tumor-purity` / 4 |
| Problematic Position Criteria | Best Peptide does not contain a problematic amino acid as defined by the `--problematic-amino-acids` parameter | `Prob Pos == None`
