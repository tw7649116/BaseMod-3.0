# BaseMod: PacBio DNA Modification Sequence Analysis

The polymerase kinetics data generated by PacBio's SMRT sequencing technology
allows simultaneous detection of common microbial base modifications in native
genomic DNA in resequencing experiments.  SMRTanalysis provides two tools for
interpreting the results: **kineticsTools** to identify modified bases, and
**MotifMaker** to detect consensus motifs associated with modified bases.
These are available in SMRTlink as an integrated pipeline.

Table of contents
=================

  * [Overview](#overview)
  * [Manual](#manual)
    * [Running with SMRTLink](#running-with-smrtlink)
    * [Running on the Command Line](#running-on-the-command-line)
    * [Running on the Command Line with pbsmrtpipe](#running-on-the-command-line-with-pbsmrtpipe)
  * [Advanced Analysis Options](#advanced-analysis-options)
    * [SMRTLink/pbsmrtpipe BaseMod Options](#smrtlinkpbsmrtpipe-basemod-options)
    * [PBAlign Options](#pbalign-options)
    * [kineticsTools Options](#kineticstools-options)
    * [MotifMaker Options](#motifmaker-options)
  * [Files](#files)
    * [PBAlign Files](#pbalign-files)
    * [kineticsTools Files](#kineticstools-files)
    * [MotifMaker Files](#motifmaker-files)
  * [Algorithm Modules](#algorithm-modules)
  * [Glossary](#glossary)

## Overview

![base modification and motif analysis](https://cloud.githubusercontent.com/assets/12494820/11598528/52b10972-9a77-11e5-9131-b1860bf3de34.png)

Analyses are performed in four stages: PBAlign, kineticsTools, MotifMaker Find, and MotifMaker Reprocess. 
* __PBAlign__
  * PBAlign maps PacBio sequences to references using an algorithm selected from a
selection of supported command-line alignment algorithms. Input can be a
fasta, pls.h5, bas.h5 or ccs.h5 file or a fofn (file of file names). Output
can be in CMP.H5, SAM or BAM format. If output is BAM format, aligner can
only be BLASR and QVs will be loaded automatically.
* __kineticsTools__
  * kineticsTools takes the AlignmentSet (mapped BAM) output of pbalign and
iterates over all nucleotides in the genome.  From the reads aligned at each
site, a profile of polymerase kinetics is obtained in the form of inter-pulse
durations (IPDs), which are compared to expectations for the given chemistry.
The ratio of observed-to-expected IPD is interpreted to decide whether a base
is modified and (optionally) determine the specific modification type.
* __MotifMaker__
  * MotifMaker is a tool for identify motifs associated with DNA
modifications in prokaryotic genomes. Modified DNA in prokaryotes
commonly arises from restriction-modification systems that methylate a
specific base in a specific sequence motif.  The canonical example the
m6A methylation of adenine in GATC contexts in E.coli. Prokaryotes may
have a very large number of active restriction-modification systems
present, leading to a complicated mixture of sequence motifs.
PacBio SMRT sequencing is sensitive to the presence of methylated DNA
at single base resolution, via shifts in the polymerase kinetics
observed in the real-time sequencing traces.  See our [publication](http://nar.oxfordjournals.org/content/40/4/e29) for
more background on modification detection.


## Manual

### Running with SMRTLink

To run Isoseq using SMRTLink, follow the usual steps for analysing data on SMRTLink. TODO: Link to document explaining SMRTLink.

### Running on the Command Line

On the command line, the analysis is performed in 4 steps:

1. Run PBAlign on your unaligned subreads, generating an aligned BAM.
2. Run kineticsTools on your aligned BAM, generating a GFF and CSV containing base modification information.
3. Run MotifMaker on your GFF generated from kineticsTools, generating a CSV containing motif information.
4. Run MotifMaker on your base modification GFF and motif CSV, generating a motif GFF.

__Step 1. PBAlign__

First, align your sequences to your chosen reference.

     pbalign --concordant --hitPolicy=randombest --minAccuracy 70 --minLength 50 --algorithmOptions="-minMatch 12 -bestn 10 -minPctIdentity 70.0" subreads.bam reference.fasta aligned_subreads.bam

Where `reference.fasta` contains your reference sequences.

Where `subreads.bam` contains your unaligned reads.

Where `aligned_subreads.bam` is where your aligned reads will be stored. 

__Step 2. kineticsTools__

Next, analyze your aligned sequences for base modifications.

     ipdSummary aligned_subreads.bam --reference reference.fasta --gff basemods.gff --csv basemods.csv --pvalue 0.001 --numWorkers 16 --identify m4C,m6A
 
**Note:** You will need to have an index file for your `reference.fasta` and for your `aligned_subreads.bam`. 

To index your `reference.fasta`, use `samtools faidx reference.fasta`. 

To index your `aligned_subreads.bam`, use `pbindex aligned_subreads.bam`. 

__Step 3. MotifMaker Find__

Next, identify consensus motifs.

     motifMaker find -f reference.fasta -g basemods.gff -o motifs.csv

__Step 4. MotifMaker Reprocess__

Finally, generate a GFF file of all modifications that are part of motifs. 

     motifMaker reprocess -f reference.fasta -g basemods.gff -m motifs.csv -o motifs.gff


### Running on the Command Line with pbsmrtpipe


####Install pbsmrtpipe
pbsmrtpipe is a part of `smrtanalysis-3.0` package and will be installed
if `smrtanalysis-3.0` has been installed on your system. Or you can [download   pbsmrtpipe](https://github.com/PacificBiosciences/pbsmrtpipe) and [install](http://pbsmrtpipe.readthedocs.org/en/master/).
    
You can verify that pbsmrtpipe is running OK by:

    pbsmrtpipe --help

#### Create a dataset
Now create an XML file from your subreads.

```
dataset create --type SubreadSet my.subreadset.xml subreads1.bam subreads2.bam ...
```
This will create a file called `my.subreadset.xml`. 


#### Create and edit basemod options and global options for `pbsmrtpipe`.
Create a global options XML file which contains SGE related, job chunking and
job distribution options that you may modify by:

```
 pbsmrtpipe show-workflow-options -o global_options.xml
```

Create a basemod options XML file which contains basemod-related options that 
you may modify by:
```
 pbsmrtpipe show-template-details pbsmrtpipe.pipelines.ds_modification_motif_analysis -o basemod_options.xml
```

The entries in the options XML files have the format:

```
 <option id="pbtranscript.task_options.min_seq_len">
            <value>300</value>
        </option>
```

And you can modify options using your favorite text editor, such as vim.

#### Run BaseMod from pbsmrtpipe
Once you have set your options, you are ready to run basemod via pbsmrtpipe:

```
pbsmrtpipe pipeline-id pbsmrtpipe.pipelines.ds_modification_motif_analysis -e eid_ref_dataset:reference.fasta -e eid_subread:my.subreadset.xml --preset-xml=basemod_options.xml --preset-xml=global_options.xml
```


## Advanced Analysis Options

### SMRTLink/pbsmrtpipe Basemod Options

| Module |    Parameter (pbsmrtpipe_name) |     Default      |  Explanation      |
| ------ | ------------------------------ | ---------------- | ----------------- |
| PBAlign | Algorithm (algorithm) | quiver  | Algorithm name |
| PBAlign | Diploid mode (diploid) | FALSE  | Enable detection of heterozygous variants (experimental) |
| PBAlign | Minimum confidence (min_confidence) | 40  | The minimum confidence for a variant call to be output to variants.gff |
| kineticsTools | Minimum coverage (min_coverage) | 5  | The minimum site coverage that must be achieved for variant calls and consensus to be calculated for a site. |
| kineticsTools | Compute methyl fraction (compute_methyl_fraction) | FALSE  | When identifying specific modifications (m4C and/or m6A), enabling this option will estimate the methylated fraction, along with 95% confidence interval bounds. |
| kineticsTools | Identify basemods (identify) |   | Specific modifications to identify (comma-separated list). Currrent options are m6A and/or m4C. |
| kineticsTools | Max sequence length (max_length) | 2112827392  | Maximum number of bases to process per contig |
| kineticsTools | P-value (pvalue) | 0.001  | P-value cutoff |
| MotifMaker | Minimum methylated fraction (min_fraction) | 30  | Minimum methylated fraction |
| MotifMaker | Minimum Qmod score (min_score) | 30  | Minimum Qmod score to use in motif finding |
| PBAlign | Algorithm options (algorithm_options) | -minMatch 12 -bestn 10 -minPctIdentity 70.0  | List of space-separated arguments passed to BLASR |
| PBAlign | Concordant alignment (concordant) | TRUE  | Map subreads of a ZMW to the same genomic location |
| pbsmrtpipe | Consolidate .bam (consolidate_aligned_bam) | FALSE  | Merge chunked/gathered .bam files |
| pbsmrtpipe | Number of .bam files (consolidate_n_files) | 1  | Number of .bam files to create in consolidate mode |
| PBAlign | Hit policy (hit_policy) | randomBest  | Specify a policy for how to treat multiple hit random : selects a random hit. all : selects all hits. allbest : selects all the best score hits. randombest: selects a random hit from all best score hits. leftmost : selects a hit which has the best score and the smallest mapping coordinate in any reference. Default value is randombest. |
| PBAlign | Min. accuracy (min_accuracy) | 70  | Minimum required alignment accuracy (percent) |
| PBAlign | Min. length (min_length) | 50  | Minimum required alignment length |
| pbsmrtpipe | Batch sort size (batch_sort_size) | 10000  | Intermediate sort size parameter (default=10000) |
| pbsmrtpipe | Force the number of regions (force_num_regions) | FALSE  | If supplied, then try to use this number (max value = 40000) of regions per reference, otherwise the coverage summary report will optimize the number of regions in the case of many references. Not compatible with a fixed region size. |
| pbsmrtpipe | Number of variants (how_many) | 100  | number of top variants to show (default=100) |
| pbsmrtpipe | Max contigs (max_contigs) | 25  | Max number of contigs to plot. Defaults to 25. |
| pbsmrtpipe | Number of regions (num_regions) | 1000  | Desired number of genome regions in the summary statistics (used for guidance, not strict). Defaults to 1000 |
| pbsmrtpipe | Region size (region_size) | 0  | If supplied, used a fixed genomic region size |

### PBAlign Options

| Type  |  Parameter          |     Example      |  Explanation      |
| ----- | ------------------ | ---------------- | ----------------- |
| positional | Input File |  unaligned.bam | SubreadSet or unaligned .bam |
| positional | Reference File |  reference.fasta | Reference DataSet or FASTA file |
| positional | Output File |  aligned.bam | Output AlignmentSet file |
| optional | Help | -h, --help | show this help message and exit |
| optional | Version | -v, --version | show program's version number and exit |
| optional | Verbose | --verbose | No Description (greg) |
| optional | Debug | --debug | Writes the debug reporting to stdout |
| optional | Profile | --profile | Print runtime profile at exit |
| optional input | Region Table | --regionTable REGIONTABLE | Specify a region table for filtering reads. |
| optional input | Configuration File | --configFile CONFIGFILE | Specify a set of user-defined argument values. |
| optional input | Pulse File | --pulseFile PULSEFILE | When input reads are in fasta format and output is a cmp.h5 this option can specify pls.h5 or bas.h5 or FOFN files from which pulse metrics can be loaded for Quiver. |
| alignment | Algorithm | --algorithm {blasr, bowtie, gmap} | Select an aligorithm from ('blasr', 'bowtie', 'gmap'). Default algorithm is blasr. |
| alignment | Maximum Hits | --maxHits MAXHITS | The maximum number of matches of each read to the reference sequence that will be evaluated. Default value is 10. |
| alignment | Minimum Anchor Size | --minAnchorSize MINANCHORSIZE | The minimum anchor size defines the length of the read that must match against the reference sequence. Default value is 12. |
| alignment | Use CCS | --useccs {useccs, useccsall, useccsdenovo} | Map the ccsSequence to the genome first, then align subreads to the interval that the CCS reads mapped to. useccs: only maps subreads that span the length of the template. useccsall: maps all subreads. useccsdenovo: maps ccs only. |
| alignment | No Split Subreads | --noSplitSubreads | Do not split reads into subreads even if subread regions are available. Default value is False. |
| alignment | Concordant | --concordant | Map subreads of a ZMW to the same genomic location. |
| alignment | Number of Threads | --nproc NPROC | Number of threads. Default value is 8. |
| alignment | Algorithm Options | --algorithmOptions ALGORITHMOPTIONS | Pass alignment options through. |
| filter criteria | Maximum Divergence | --maxDivergence MAXDIVERGENCE | The maximum allowed percentage divergence of a read from the reference sequence. Default value is 30.0. |
| filter criteria | Minimum Accuracy | --minAccuracy MINACCURACY | The minimum percentage accuracy of alignments that will be evaluated. Default value is 70.0. |
| filter criteria | Minimum Length | --minLength MINLENGTH | The minimum aligned read length of alignments that will be evaluated. Default value is 50. |
| filter criteria | Score Cutoff | --scoreCutoff SCORECUTOFF | The worst score to output an alignment. |
| filter criteria | Hit Policy | --hitPolicy {randombest, allbest, random, all, leftmost} | Specify a policy for how to treat multiple hit random: selects a random hit. all: selects all hits. allbest: selects all the best score hits. randombest: selects a random hit from all best score hits. leftmost: selects a hit which has the best score and the smallest mapping coordinate in any reference. Default value is randombest. |
| filter criteria | Filter Adapter-Only | --filterAdapterOnly |  If specified, do not report adapter-only hits using annotations with the reference entry. |
| for cmp.h5 | For Quiver | --forQuiver | The output cmp.h5 file which will be sorted, loaded with pulse QV information, and repacked, so that it can be consumed by quiver directly. This requires the input file to be in PacBio bas/pls.h5 format, and --useccs must be None. Default value is False. |
| for cmp.h5 | Load QVs | --loadQVs | Similar to --forQuiver, the only difference is that --useccs can be specified. Default value is False. |
| for cmp.h5 | By Read | --byread | Load pulse information using -byread option instead of -bymetric. Only works when --forQuiver or --loadQVs are set. Default value is False. |
| for cmp.h5 | Metrics | --metrics METRICS | Load the specified (comma-delimited list of) metrics instead of the default metrics required by quiver. This option only works when --forQuiver  or --loadQVs are set. Default: DeletionQV, DeletionTag, InsertionQV, MergeQV, SubstitutionQV |
| miscellaneous | Seed | --seed SEED | Initialize the random number generator with a none-zero integer. Zero means that current system time is used. Default value is 1. |
| miscellaneous | Temporary Directory | --tmpDir TMPDIR | Specify a directory for saving temporary files. Default is /scratch. |

### kineticsTools Options

|  Parameter          |     Example      |  Explanation      |
| ------------------ | ---------------- | ----------------- |
| Input Reads | aligned_subreads.bam | This is a positional argument and the first argument to be specified. Can be a BAM or Alignment DataSet. |
| Reference File | --reference reference.fasta | Fasta or Reference DataSet (default: None) |
| Modifications GFF | --gff basemod.gff | Output GFF file of modified bases (default: None) |
| Modifications CSV | --csv basemod.csv | Output CSV file out per-nucleotide information (default: None) |
| Help |  -h, --help | show this help message and exit |
| Version | -v, --version | show program's version number and exit |
| Log Level | --log-level {DEBUG, INFO, WARNING, ERROR, CRITICAL} | Set log level (default: INFO) |
| Verbose | --verbose | Print verbose job information to stdout | 
| Debug Mode | --debug | Debug to stdout (default: False) |
| Number of Jobs | --numWorkers NUMWORKERS, -j NUMWORKERS | Number of thread to use (-1 uses all logical cpus) (default: 1) |
| P-Value | --pvalue PVALUE | Kinetic Deviation P-value cutoff (default: 0.01) (nat) |
| Maximum Length | --maxLength MAXLENGTH | Maximum number of bases to process per contig, the default is 3 trillion. (default: 3000000000000)  |
| Identify |  --identify IDENTIFY | Specific modifications to identify (comma-separated list). Currrent options are m6A, m4C, m5C_TET. Cannot be used with --control. (default: ) |
| Compute Methyl Fraction | --methylFraction | In the --identify mode, add --methylFraction to command line to estimate the methylated fraction, along with 95% confidence interval bounds. (default: False) |
| Output Files | --outfile OUTFILE | Use this option to generate all possible output files. Argument here is the root filename of the output files. (default: None) |
| m5C Scores | --m5Cgff M5CGFF | Name of output GFF file containing m5C scores (default: None) |
| m5C Classifier | --m5Cclassifer M5CCLASSIFIER | Specify csv file containing a 127 x 2 matrix (default: None) |
| CSV H5 Format | -csv_h5 CSV_H5 | Name of csv output to be written in hdf5 format. (default: None) |
| Summary H5 | --summary_h5 SUMMARY_H5 | Name of output summary h5 file. (default: None) |
| Multisite Detection | --ms_csv MS_CSV | Multisite detection CSV file. (default: None) |
| Case-Control Mode | --control CONTROL  | cmph.h5 file containing a control sample. Tool will perform a case-control analysis (default: None) |
| LDA | --useLDA | Set this flag to debug LDA for m5C/Ca5C detection (default: False) |
| Minimum Call Coverage | --minCoverage MINCOVERAGE | Minimum coverage required to call a modified base (default: 3) |
| Maxmimum Queue Size | --maxQueueSize MAXQUEUESIZE | Max Queue Size (default: 20) |
| Maximum Coverage | --maxCoverage MAXCOVERAGE | Maximum coverage to use at each site (default: -1) |
| MapQV Threshold | --mapQvThreshold MAPQVTHRESHOLD | No Description (nat) |
| IPD Model | --ipdModel IPDMODEL | Alternate synthetic IPD model HDF5 file (default: None) |
| IPD Percentile Cap | --cap_percentile CAP_PERCENTILE | Global IPD percentile to cap IPDs at (default: 99.0) |
| Minimum MethylFraction Coverage | --methylMinCov METHYLMINCOV | Do not try to estimate methylFraction unless coverage is at least this. (default: 10) |
| Minimum Modification Coverage | --identifyMinCov IDENTIFYMINCOV | Do not try to identify the modification type unless coverage is at least this. (default: 5) |
| Maximum Alignments |  --maxAlignments MAXALIGNMENTS | Maximum number of alignments to use for a given window (default: 1500) |
| Reference Window |  -w REFERENCEWINDOWSASSTRING, --referenceWindow REFERENCEWINDOWSASSTRING, --referenceWindows REFERENCEWINDOWSASSTRING, --refContigs REFERENCEWINDOWSASSTRING | The window (or multiple comma-delimited windows) of the reference to be processed, in the format refGroup [:refStart-refEnd] (default: entire reference). (default: None) |
| Reference Window File | -W REFERENCEWINDOWSASSTRING, --referenceWindowsFile REFERENCEWINDOWSASSTRING, --refContigsFile REFERENCEWINDOWSASSTRING | A file containing reference window designations, one per line (default: None) |
| Skip Unregognized Contigs | --skipUnrecognizedContigs SKIPUNRECOGNIZEDCONTIGS | Whether to skip, or abort, unrecognized contigs in the -w/-W flags (default: False) |
| Alignment Window | --alignmentSetRefWindows | DEVELOPER OPTION |
| Reference Contig Index | --refContigIndex REFCONTIGINDEX | DEVELOPER OPTION |
| Model Iterations | --modelIters MODELITERS | DEVELOPER OPTION |
| IPD parameters | --paramsPath PARAMSPATH | DEVELOPER OPTION |
| Pickle | --pickle PICKLE | DEVELOPER OPTION |
| Thread | --threaded, -T | DEVELOPER OPTION |
| Profile | --profile  | DEVELOPER OPTION |
| PBD Debugger | --usePdb | DEVELOPER OPTION |
| Seed | --seed RANDOMSEED | DEVELOPER OPTION |
| Emit Tool Contract | --emit-tool-contract | DEVELOPER OPTION |
| Resolved Tool Contract | --resolved-tool-contract RESOLVED_TOOL_CONTRACT | DEVELOPER OPTION |

### MotifMaker Options

MotifMaker options, except for `--help`, require the specification of a command first- either `find` or `reprocess`. 

| Command  |  Parameter          |     Example      |  Explanation      |
| ----- | ------------------ | ---------------- | ----------------- |
| | Help | -h, --help | show this help message and exit |
| find | Reference FASTA |  -f reference.fasta, --fasta reference.fasta | Reference fasta file |
| find | Modifications GFF | -g basemod.gff, --gff basemod.gff | basemod.gff or .gff.gz file |
| find | Minimum Qmod Score |  -m SCORE, --minScore SCORE | Minimum Qmod score to use in motif finding (default: 30.0) |
| find | Motif CSV | -o motifs.csv, --output motifs.csv | Output motifs csv file |
| find | Parallelize | -p, --parallelize | Parallelize motif finder (default: true) |
| find | Motif XML | -x motifs.xml, --xml motifs.xml | Output motifs xml file |
| reprocess | Modifications CSV | -c basemod.csv, --csv basemod.csv |  Raw basemod.csv file |
| reprocess | Reference FASTA | -f reference.fasta, --fasta reference.fasta |  Reference fasta file |
| reprocess | Modifications GFF | -g basemod.gff, --gff basemod.gff|  original basemod.gff or .gff.gz file |
| reprocess | Minimum MethylFraction | --minFraction FRACTION |  Only use motifs above this methylated fraction (default: 0.0) |
| reprocess | Motifs CSV | -m motifs.csv, --motifs motifs.csv|  motifs csv |
| reprocess | Motifs GFF | -o motifs.gff, --output motifs.gff|  Reprocessed basemod.gff with motif information added |

## Files

### PBAlign Files

#### Inputs

__Unaligned Reads File (subreads.\*)__

(Required) This file contains your input reads. It can be a BAM, FASTA, XML, CCS.H5 or BAS.H5 file.

__Reference File(reference.fasta)__

(Required) This file contains the reference sequences that your input reads will be aligned against.

__Configuration File__

(Optional) If you prefer to specify all arguments in a config file rather than on the command line, use the `--configFile` argument.

Example config text file:
```
# This is a config file for pbalign where users can specify
# values for an arbitary subset of optional arguments for pbalign.
# Lines which start with '#' are comments. Otherwise each line
# should specify the value for exactly one optinal argument.
# Note that:
#   [1] Positional arguments, including inputFileName, outputFileName,
#       and referencePath, can not be specified in a config file.
#   [2] Sepcial arguments, including --verbose, --version (-v..),
#       --debug and --profile, in a config file will be ignored.
#   [3] Arguments specified in a config file will be overwritten
#       by arguments on command-line.

# Aligner's options.
--maxHits       = 20
--minAnchorSize = 1
--minLength     = 100
--algorithmOptions = "-noSplitSubreads -maxMatch 30 -nCandidates 30"

# SamFilter's filtering criteria and hit policy.
--hitPolicy     = randombest
--maxDivergence = 40

# Miscellaneous
--seed = 10
```

__Pulse File__

(Optional) Path to \*.bas.h5 or \*.pls.h5 when input type is \*.fasta, and output type is \*.cmp.h5. This is necessary to load pulse metrics into the alignment file (\*.cmp.h5) for subsequent consumption by `variantCaller`.

__Region Table__

(Optional) path to \*.rgn.h5. When input file is of type \*.bas.h5, you may supply an optional Region table (\*.rgn.h5) to filter the reads prior to alignment

####Outputs

__Aligned Reads File (aligned_subreads.\*)__

This file contains your aligned output reads. You can specify \*.cmp.h5, \*.sam or \*.bam
Note: \*.bam output format only works if `--algorthim blasr`


### kineticsTools Files

__Modifications CSV File (basemods.csv)__

The modifications.csv file contains one row for each (reference position, strand) pair that appeared in the dataset with coverage at least x. x defaults to 3, but is configurable with ‘–minCoverage’ flag to ipdSummary.py. The reference position index is 1-based for compatibility with the gff file the R environment.

The output columns vary depending on whether `--control` was used to run in case-control mode, or whether the program ran in the default in-silico control mode.

**in-silico control mode output columns**

|Column	| Description |
| ----- | ----------- |
|refId	| reference sequence ID of this observation |
tpl		| 1-based template position |
strand		| native sample strand where kinetics were generated. ‘0’ is the strand of the original FASTA, ‘1’ is opposite strand from FASTA	| 
base		| the cognate base at this position in the reference	| 
score		| Phred-transformed pvalue that a kinetic deviation exists at this position	| 
tMean		| capped mean of normalized IPDs observed at this position	| 
tErr		| capped standard error of normalized IPDs observed at this position (standard deviation / sqrt(coverage)	| 
modelPrediction		| normalized mean IPD predicted by the synthetic control model for this sequence context	| 
ipdRatio		| tMean / modelPrediction	| 
coverage		| count of valid IPDs at this position (see Filtering section for details)	| 
frac		| estimate of the fraction of molecules that carry the modification	| 
fracLow		| 2.5% confidence bound of frac estimate	| 
fracUpp		| 97.5% confidence bound of frac estimate	| 

**case-control mode output columns**

|Column	| Description |
| ----- | ----------- |
|refId		| reference sequence ID of this observation|
|tpl		| 1-based template position|
|strand		| native sample strand where kinetics were generated. ‘0’ is the strand of the original FASTA, ‘1’ is opposite strand from FASTA|
|base		| the cognate base at this position in the reference|
|score		| Phred-transformed pvalue that a kinetic deviation exists at this position|
|caseMean		| mean of normalized case IPDs observed at this position|
|controlMean		| mean of normalized control IPDs observed at this position|
|caseStd		| standard deviation of case IPDs observed at this position|
|controlStd		| standard deviation of control IPDs observed at this position|
|ipdRatio		| tMean / modelPrediction|
|testStatistic		| t-test statistic|
|coverage		| mean of case and control coverage|
|controlCoverage		| count of valid control IPDs at this position (see Filtering section for details)|
|caseCoverage		| count of valid case IPDs at this position (see Filtering section for details)|

__Modifications GFF File (basemods.gff)__

The modifications.gff is compliant with the GFF Version 3 [specification](http://www.sequenceontology.org/gff3.shtml). Each template position / strand pair whose p-value exceeds the pvalue threshold appears as a row. The template position is 1-based, per the GFF spec. The strand column refers to the strand carrying the detected modification, which is the opposite strand from those used to detect the modification. The GFF confidence column is a Phred-transformed pvalue of detection.

The modifications.gff file will not work directly with most genome browsers. You will likely need to make a copy of the GFF file and convert the _seqid_ columns from the generic ‘ref0000x’ names generated by PacBio, to the FASTA headers present in the original reference FASTA file. The mapping table is written in the header of the modifications.gff file in #sequence-header tags. This issue will be resolved in the 1.4 release of kineticsTools.

The auxiliary data column of the GFF file contains other statistics which may be useful downstream analysis or filtering. In particular the coverage level of the reads used to make the call, and +/- 20bp sequence context surrounding the site.

|Column	| Description |
| ----- | ----------- |
|seqid	 | Fasta contig name|
|source	 | Name of tool – ‘kinModCall’|
|type	 | Modification type – in identification mode this will be m6A, m4C, or m5C for identified bases, or the generic tag ‘modified_base’ if a kinetic event was detected that does not match a known modification signature|
|start	 | Modification position on contig|
|end	 | Modification position on contig|
|score	 | Phred transformed p-value of detection - this is the single-site detection p-value|
|strand	 | Sample strand containing modification|
|phase	 | Not applicable|
|attributes	 | Extra fields relevant to base mods. IPDRatio is traditional IPDRatio, context is the reference sequence -20bp to +20bp around the modification, and coverage level is the number of IPD observations used after Mapping QV filtering and accuracy filtering. If the row results from an identified modification we also include an identificationQv tag with the from the modification identification procedure. identificationQv is the phred-transformed probability of an incorrect identification, for bases that were identified as having a particular modification. frac, fracLow, fracUp are the estimated fraction of molecules carrying the modification, and the 5% confidence intervals of the estimate. The methylated fraction estimation is a beta-level feature, and should only be used for exploratory purposes. |

### MotifMaker Files

__Motifs GFF File (motifs.gff)__

This is a reprocessed version of modifications.gff with the following changes. If a detected modification occurs on a motif detected by the motif finder, the modification is annotated with motif data. An attribute ‘motif’ is added containing the motif string, and an attribute ‘id’ is added containing the motif id, which is the motif string for unpaired motifs or ‘motifString1/motifString2’ for paired motifs. If a motif instance exists in the genome, but was not detected in modifications.gff, an entry is added to motifs.gff, indicating the presence of that motif and the kinetics that were observed at that site.

__Motifs CSV File (motifs.csv)__

This file summarizes the modified motifs discovered by the tool. The CSV contains one row per detected motif, with the following columns.

|Column	| Description |
| ----- | ----------- |
|motifString	| Detected motif sequence| 
|centerPos	| Position in motif of modification (0-based)| 
|fraction	| Fraction of instances of this motif with modification QV above the QV threshold| 
|nDetected	| Number of instances of this motif with above threshold| 
|nGenome	| Number of instances of this motif in reference sequence| 
|groupTag	| A string identifying the motif grouping. For paired motifs this is “<motifString1>/<motifString2>”, For unpaired motifs  this equals motifString| 
|partnerMotifString	| motifString of paired motif (motif with reverse-complementary motifString)| 
|meanScore	| Mean Modification Qv of detected instances| 
|meanIpdRatio	| Mean IPD ratio of detected instances| 
|meanCoverage	| Mean coverage of detected instances| 
|objectiveScore	| Objective score of this motif in the motif finder algorithm| 

## Algorithm Modules

### BLASR

BLASR is a read mapping program that maps reads to positions in a genome by clustering short exact matches between the read and the genome, and scoring clusters using alignment. The matches are generated by searching all suffixes of a read against the genome using a suffix array. Global chaining methods are used to score clusters of matches. It is exremely useful to have read filtering information, and mapping runtime may decrease substantially when a precomputed suffix array index on the reference sequence is specified. 

### kineticsTools

##### Synthetic Control

Studies of the relationship between IPD and sequence context reveal that most of the variation in mean IPD across a genome can be predicted from a 12-base sequence context surrounding the active site of the DNA polymerase. The bounds of the relevant context window correspond to the window of DNA in contact with the polymerase, as seen in DNA/polymerase crystal structures. To simplify the process of finding DNA modifications with PacBio data, the tool includes a pre-trained lookup table mapping 12-mer DNA sequences to mean IPDs observed in C2 chemistry.

##### Filtering and Trimming

kineticsTools uses the Mapping QV generated by BLASR and stored in the BAM file to ignore reads that aren’t confidently mapped. The default minimum Mapping QV required is 10, implying that BLASR has 90% confidence that the read is correctly mapped. Because of the range of read lengths inherent in PacBio data. This can be changed in using the `–mapQvThreshold` command line argument, or via the SMRT Link configuration dialog for Modification Detection.

There are a few features of PacBio data that require special attention in order to acheive good modification detection performance. kineticsTools inspects the alignment between the observed bases and the reference sequence – in order for an IPD measurement to be included in the analysis, the PacBio read sequence must match the reference sequence for k around the cognate base. In the current module k=1, the IPD distribution at some locus be thought of as a mixture between the ‘normal’ incorporation process IPD, which is sensitive to the local sequence context and DNA modifications and a contaminating ‘pause’ process IPD which have a much longer duration (mean >10x longer than normal), but happen rarely (~1% of IPDs). Note: Our current understanding is that pauses do not carry useful information about the methylation state of the DNA, however a more careful analysis may be warranted. Also note that modifications that drastically increase the Roughly 1% of observed IPDs are generated by pause events. Capping observed IPDs at the global 99th percentile is motivated by theory from robust hypothesis testing. Some sequence contexts may have naturally longer IPDs, to avoid capping too much data at those contexts, the cap threshold is adjusted per context as follows: capThreshold = max(global99, 5*modelPrediction, percentile(ipdObservations, 75))

##### Statistical Testing

We test the hypothesis that IPDs observed at a particular locus in the sample have a longer means than IPDs observed at the same locus in unmodified DNA. If we have generated a Whole Genome Amplified dataset, which removes DNA modifications, we use a case-control, two-sample t-test. This tool also provides a pre-calibrated ‘synthetic control’ model which predicts the unmodified IPD, given a 12 base sequence context. In the synthetic control case we use a one-sample t-test, with an adjustment to account for error in the synthetic control model.

### MotifMaker

Existing motif finding algorithms such as MEME-chip and YMF are
sub-optimal for this case for the following reasons:
 
1. They search for a single motif, rather than attempting to identify a
   complicated mixture of motifs
 
2. They generally don't accept the notion of aligned motifs - the
   input to the tools is a window into the reference sequence which
   can contain the motif at any offset, rather a single center
   position that is available with kinetic modification detection.
 
3. Implementations generally either use a Markov model of the
   reference (MEME-chip), or do exact counting on the reference, but
   place restrictions on the size and complexity of the motifs that
   can be discovered.
 
 
Here we give a rough overview of the algorthim used by
MotifMaker.

Define a motif as a set of (position relative to
methylation, required base) tuples. Positions not listed in the motif
are implicitly degenerate.  Given a list of modification detections
and a genome sequence, we define the following objective function on
motifs:
 
```
Motif score(motif) = 
(# of detections matching motif) / (# of genome sites matching motif)  *  (Sum of log-pvalue of detections matching motif) = 
(fraction methylated) * (sum of log-pvalues of matches)
 ```
 
We search (close to exhaustively) through the space of all possible
motifs, progressively testing longer motifs using a branch-and-bound
search. The 'fraction methylated' term must be less than 1, so the
maximum achievable score of a child node is the sum of scores of
modification hits in the current node, permitting us to prune all
search paths whose maximum achievable score is less than the best
score discovered so far.


## Glossary

 * __Inter-Pulse Duration (IPD)__
 * In the terminology of SMRT sequencing, pulses are single-molecule
   fluorescence signals interpreted as basecalls.  The IPD for a basecall is
   the time between the current pulse and the next one.  Many covalent base
   modifications cause a temporary pause in the sequencing reaction, which
   varies depending on the type of modification.  For detecting basemods,
   kineticsTools uses the IPD averaged over many subreads aligned at the same
   position.

<sup>For Research Use Only. Not for use in diagnostic procedures. © Copyright 2015, Pacific Biosciences of California, Inc. All rights reserved. Information in this document is subject to change without notice. Pacific Biosciences assumes no responsibility for any errors or omissions in this document. Certain notices, terms, conditions and/or use restrictions may pertain to your use of Pacific Biosciences products and/or third party products. Please refer to the applicable Pacific Biosciences Terms and Conditions of Sale and the applicable license terms at http://www.pacificbiosciences.com/licenses.html.</sup>

<sup> Visit the [PacBio Developer's Network Website](http://pacbiodevnet.com) for the most up-to-date links to downloads, documentation and more. </sup>

<sup>[Terms of Use & Trademarks](http://www.pacb.com/legal-and-trademarks/site-usage/) | [Contact Us](mailto:devnet@pacificbiosciences.com) </sup>
