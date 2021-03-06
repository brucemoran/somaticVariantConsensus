# somaticVariantConsensus
## A method for consensus SNV calls allowing multiple callers
## (Also makes plots)

### How to:

**Required Inputs**
- VCFs (at least two or the method is pointless) annotated by VEP using below flags:
 ```
 vep --dir_cache $CACHEDIR \
   --offline \
   --assembly GRCh37 \
   --vcf_info_field ANN \
   --symbol \
   --species homo_sapiens \
   --check_existing \
   --cache \
   --merged \
   --fork $FORK \
   --af_1kg \
   --af_gnomad \
   --vcf \
   --input_file $VCF \
   --output_file $VCFOUT \
   --format "vcf" \
   --fasta $GENOME \
   --hgvs \
   --canonical \
   --ccds \
   --force_overwrite \
   --verbose
 ```
- R libraries. Install as below, or use the Singularity container
 ```
 library("BiocManager")

 #packages
 libs <- c("customProDB",
      	   "ensemblVEP",
      	   "GenomicRanges",
      	   "org.Hs.eg.db",
           "data.table",
       	   "pheatmap",
      	   "tidyverse")

 lapply(libs, BiocManager::install(dependencies = TRUE, update = TRUE, ask = FALSE))
 ```

- Variables for R scripts
 ```
 CALLDIR = <directory where your VCFs reside, NB name of this dir is used to tag filenames (best use case: dir named after caller>
 SCRIPTDIR = <where you put the scripts> from the ./scripts of this repo
 WORKDIR = <directory in which to save RData GRanges of variant calls used in rescue, plotting>
 VEPVCFPATTERN = <file pattern of VCF files annotated by VEP, use end extension, this is for input so one per sample>
 RAWVCFPATTERN = <as above but for unannotated, unfiltered 'raw' calls>
 GERMLINE = <name of germline sample used to call against somatic, used to find somatic sample column in VCFs>
 TAG = <string with which to tag outputs>
 INCLUDEDORDER = <string of comma-delimited order in which your samples whould be plotted (heatmap of frequency across samples>
 ```

**Running**
- Make RData containing GRanges of multiple samples from VEP-annotated and raw VCFs
 Following variant calling with CallerX which is in directory structure: /data/project/analysis/WGS/calls/CallerX
 ```
 Rscript --vanilla $SCRIPTDIR/variants_GRanges_consensus_plot.func.R \
    $RLIBPATH \
    $SCRIPTDIR \
    $CALLDIR \
    $WORKDIR \
    $GERMLINE \
    $VEPVCFPATTERN \
    $RAWVCFPATTERN
 ```
 This outputs an RData file containing calls for all samples with the $VEPVCFPATTERN, $RAWVCFPATTERN (e.g. sets of files with extensions 'vep90.snv.vcf', 'raw.snv.vcf' will be parsed into GRanges and saved); these are saved to $WORKDIR

 N.B. that the actual format of the RData is a list of GRanges (NOT GRangesList), so access is as per functions herein, read the code to see how it is performed.

- Rescue somatic calls, plot
 Directly after the above, run:
 ```
 Rscript --vanilla $SCRIPTDIR/SNV_GRanges_consensus_plot.caller.R \
    $RLIBPATH \
    $SCRIPTDIR \
    $WORKDIR \
    $TAG \
    $INCLUDEDORDER
 ```
 This returns tables of variants per sample (to put in Excel sheets for your colleagues) and plots of shared and shared+private variants. A tumour mutational burden function is also in place, based on Illumina's Rapid Nextera kit. The result is returned within the filename of the samples for the 'ALL' impact outputs.

**Output**
- RData GRanges objects
 Given three samples named Tumour, Recurrence, Metastasis; SNVs called with MuTect2 and Strelka2, you have 12 VCFs as input: Tumour.MuTect2.snv.raw.vcf, Tumour.Strelka2.snv.raw.vcf, Tumour.MuTect2.snv.vep90.vcf, etc.
 There are four RData made in the first instance by vepVCF_to_GRangesRData.caller.R: MuTect2.snv.raw.RData, MuTect2.snv.vep90.RData, Strelka2.snv.raw.RData, Strelka2.snv.vep90.RData, each is set up internally like this:
 ```
 > MuTect2_GRanges <- get(load("MuTect2.vep90.RData"))
 > names(MuTect2_GRanges)
 [1] Metastasis Recurrence Tumour
 ```
 NB that this was set up for multiple variant types, so supports indels, SVs in terms of holding that information.
 Running SNV_GRanges_consensus_plot.caller.R, the first thing that happens is to load the sets of RData for variant types. Then in subsequent functions, these are get()'d and processed.
 Errors will arise if $CALLDIR contains any further RData objects.

- Tables
 Header =  seqnames, start, end, width, strand, AD, AD.1, AF, Consequence, IMPACT, SYMBOL, HGVSc, HGVSp, HGVSp1
 AD, AD.1 = Somatic, Germline
 HGVSp, HGVSp1 = 3-letter, 1-letter annotations (3-letter, produced by VEP, is converted to 1-letter internally)
 Rest are self explanatory IMHO

- Plots
 Two produced: 'only-shared' and 'private-shared' per variant-type, and per IMPACT ('HM' -> 'high' + 'moderate'; ALL -> all IMPACTs). Total per run is 4 per variant type.
 Idea was to represent all mutations shared between any samples, and to visualise the extent of variants across all samples. NB samples can be removed by excluding from $INCLUDEDORDER.
 Examples:
 [only-shared](https://github.com/brucemoran/rescueSomaticVariantCalls/blob/master/images/HM.snv.only-shared.consensus.pdf)
 NB the shared *PTEN* and *RB1* mutations (common across high-grade ovarian disease)
 [private-shared](https://github.com/brucemoran/rescueSomaticVariantCalls/blob/master/images/HM.snv.private-shared.consensus.pdf)
 NB the extent of mutation in 'Normal' which was taken following 3 cycles of carboplatin, a potent mutagen

**To-do**
 - [ ] Scale bar for private-shared to show extent.
 - [ ] Print TMB values to private-shared.
