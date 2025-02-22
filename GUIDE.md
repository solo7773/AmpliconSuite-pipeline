## Primer on running AmpliconArchitect

### Contents
 - [What is AA?](#what-is-aa)
 - [Preparing the inputs](#preparing-the-inputs)
 - [Selecting seed regions](#filtering-and-selecting-seeds-from-cnv-data)
 - [Worked example](#worked-example)
 - [FAQ](#faq)

#

### What is AA?
AmpliconArchitect (AA) is a tool designed to study **focal amplifications** in the cancer genome. AA can help identify common sources of focal amplifications, such as **circular extrachromosomal DNA (ecDNA)**, and **breakage-fusion-bridge cycles (BFBs)**.
 
AA takes as input WGS data and a bed file of regions to examine, and outputs a **breakpoint graph** and a **cycles file**. The **breakpoint graph**
encodes the identity of regions in the genome, their copy numbers (as measured by AA), and the breakpoint junctions between
these elements (as measured by AA). The **cycles file** outputs both non-cyclic paths and cyclic paths which are decompositions
of the breakpoint graph into paths which best explain the CN of the segments.


An open-access publication detailing AA's methodology is available in [Nature Communications](https://www.nature.com/articles/s41467-018-08200-y). Please cite AA if you use it in your work. 
#

### Installing AA
Please refer to the [AA README](https://github.com/virajbdeshpande/AmpliconArchitect#installation) for complete 
installation instructions.

You may also want to install the additional programs for classifying outputs ([AmpliconClassifier](https://github.com/jluebeck/AmpliconClassifier)) and visualization ([CycleViz](https://github.com/jluebeck/CycleViz)).
#

### Preparing the inputs
![AA workflow](./images/AA_example.png)

AA takes as input a WGS BAM file (paired-end WGS), and a user-created BED file of seed regions as inputs. Here we will discuss some of 
the best practices for generating these files.

AA uses external CNV calls to determine which regions it should examine - thes are called **CNV seeds**. 
However, AA independently calls copy number inside the seed regions it is tasked with, and thus after selecting the regions, 
**the CN estimates are not propogated into AA's own estimations.**

##### <ins>To help standardize the process of running AA, we have created a wrapper tool, called [AmpliconSuite-pipeline](https://github.com/jluebeck/PrepareAA) </ins> 

AmpliconSuite-pipeline wraps the required steps before running AA. Users will enter the AA workflow from different stages. Some will start with .fastq files, others will have a BAM file only, and others will
already have the BAM file and CNV seed regions they wish to analyze. We have created this wrapper to allow users to enter 
the workflow from any point. AmpliconSuite-pipeline wraps BWA MEM for alignment, CNVKit for CNV seed identification, and will also invoke
the AA `amplified_intervals.py` filtering script to select/filter/merge CNV seeds which are appropriate for AA.

Ultimately, AmpliconSuite-pipeline can even invoke AA (if installed beforehand), and thus saves users from the hassle associated with preparing everything to run AA on a sample.

If you decide to "got it alone" and not to use AmpliconSuite-pipeline, please carefully read the following points:

#### - Creating the BAM file:
If you are generating your own BAM file, please note that aligners vary in terms of which tags they will add to BAM files. Furthermore,
BAM files hosted on SRA are often stripped of tags needed by AA to correctly identify breakpoints. At this time, we recommend using
**BWA MEM**. However, we have also modified AA so that it will support BAM files created by Illumina's Isaac aligner. Please note that for use with AA, BWA MEM should be run **without** setting the `-m` flag (it's off by default, just don't turn it on).

Please also note that the AA data repo has reference genome fasta files you can align to. We recommend them as they are stripped of alternate contigs, and thus may cause fewer issues with CN estimation in AA.

Don't forget to also create the BAM file index! `samtools index [sample].bam`

#### - Creating the CNV bed file:
It is important to remember that the goal of the CNV calls given to AA are simply to identify locations where focal amplifications may exist - that is, there is a locally increased copy number. This means that CNV callers which sensitively segment the reference genome will perform best for focal amplification detection. CNV callers that undersegment the genome may have good performance in genome-wide tool comparisons, but may not work well with AA.

In the AA publication, ReadDepth was used as the CNV caller for seeding. However, there are much more modern CNV callers available. Some other callers we have used successfully include CNVKit, Canvas, Battenberg, and GATK.

If you generate your own file of CNV calls, *please ensure the estimated copy number of these calls is in the last column of the bed file*. 
Secondly, also ensure that the calls you are using are segmented, and not just raw per-bin estimates. This is not a concern for most users,
however **if you notice there are > 50 CNV seeds going into AA, there may be something wrong**.   

We have found that CNV callers FACETS and ASCAT generally undersegment or create seeds that are too long for AA. **We do not recommend using FACETS or ASCAT for AA seed detection.**

Many modern callers are purity and ploidy aware. Note that while this may help to accurate get globally accurate copy numbers, it can lead to situations where entire chromosome arms surpass your copy number gain threshold for AA! If you are using purity and ploidy corrected calls, please set the `gain` cutoff accordingly.
#

### Filtering and selecting seeds from CNV data
Focal amplifications are somewhat aribtrarily defined as regions of the genome with a CN > 5 and 10 kbp < size < 10 Mbp.

**Selecting appropriate seed regions from the CNV estimates is critical to properly running AA**.

We recommend picking regions which have an estimated CN >= 4.5 and size > 50 kbp, which do not appear amplified due to being parts of repeat elements, and which are not amplified due to karyotypic abnormality. 

PrepareAA.py calls on multiple filters to ensure that CNV seeds are appropriately selected. Attempting to bypass this filtering and implement some alternative or less rigorous strategy is typically detrimental to getting high-quality focal amplification calls.

CNV estimates can be imperfect in low-complexity or repetitive regions or appear consistently high when there is karyotypic abnormality. Thus, we have developed modules called `cnv_prefilter.py` and `amplified_intervals.py` to address those issues. They are used by default.

If low-complexity/repetive seeds are not filtered from AA, it can cause an exremely long runtime and produce results which are not useful. AA has its own filters for these regions, but it should still be avoided to give them to AA as input.

If you have CNV segments which are > 10 Mbp, we suggest you run the `seed_trimmer.py` script in the AmpliconSuite-pipeline/scripts directory (and documented in AmpliconSuite-pipeline's repo). This will pre-filter some regions of low mappability, conserved CNV gain, etc. 
The output of this script can then be fed to `amplified_intervals.py`. Invoking AA via AmpliconSuite-pipeline will yield the best results.
#

### Constructing unified seed sets from multiple related samples
One common type of analysis is the detection of focal amplifications in samples derived from the same patient or cell line. These may include multiple samples derived from multiregion biopsies, timecourse sequencing on the same individual, or analysis of a cell line before and after drug treatment.

**To maximize the utility of AA on samples from the same source, you should consider merging your AA seed files from related samples.**
We will work on automating this process in the future, but these are the instructions users should follow for now.

#### Option A: `GroupedAnalysis.py` (recommended)
This can be done using the script `GroupedAnalysis.py`. This assumes you are starting from .bam files. Simply place your samples into a three column file, formatted like so 
>`sample_name` `path/to/sample.bam`  `'Tumor' or 'Normal'`

Users can provide two additional columns
> `/path/to/CNV.bed` `/path/to/sample_metadata.json`

These are positional for columns 4 and 5, respectively. If one is ommitted, the `NA` should be placed.

`GroupedAnalysis.py` takes most of the same arguments as `PrepareAA.py`, but by default `--run_AA` and `--run_AC` will be set.

#### Option B: Manual creation of unified seeds
First run AmpliconSuite-pipeline on each sample **without** setting `--run_AA`. The result for each sample the `[sample]_AA_CNV_SEEDS.bed` file. Next, using `bedtools` or similar, take the relevant seeds files from related samples and merge those bed files. A tutorial for merging bed files can be found [here](http://quinlanlab.org/tutorials/bedtools/bedtools.html) 
(see section entitled "bedtools 'merge'").

Next, place a placeholder copy number estimate into the last column of each row. Since the regions are already filtered, AA does not need to consider the copy number estimate before running its own CN-estimation on each sample.
E.g. `sed -i "s/$/\t999999/" initial_merged_AA_CNV_SEEDS.bed > final_merged_AA_CNV_SEEDS.bed`

The resulting merged file should still end with the suffix `_AA_CNV_SEEDS.bed`, since this suffix has a special meaning in AmpliconSuite-pipeline, and filtering will be skipped (these samples are already filtered.)

You can then run `PrepareAA.py` with this merged `AA_CNV_SEEDS.bed` file for each of the related samples, now ensuring that each sample is launched on the same collection of regions.
#

### Running AA
We assume the user now has a coordinate-sorted BAM file, and a CNV seed BED file (i.e., a BED file of seeds output by AmpliconSuite-pipeline `amplified_intervals.py`).
To check if your BAM file is coordinate-sorted, you can take a peek at the BAM file header by doing 
`samtools view -H your_bamfile.bam | head `. 
Please make sure you know which reference genome it is aligned to so that you can properly specify the `--ref` argument to AA.

For more on running AA, please see the [relevant section of the AA README](https://github.com/virajbdeshpande/AmpliconArchitect#running-ampliconarchitect) or jump down to the [worked example](#worked-example).

Note that AmpliconSuite-pipeline can run AA on its own by setting `--run_AA`, and AA will automatically be called at the end of the preparation process without any additional work by the user.
#

### Interpreting the output
"*How do I know if my sample has ecDNA (or BFB, or other)?*"

**We have recently developed classification methods which take AA output and predict the mechanism(s) of a focal amplification's genesis**.
This method is called **[AmpliconClassifier](https://github.com/jluebeck/AmpliconClassifier)**. 
This tool will output a table describing many important details of the focal amplifications AA discovered. AmpliconClassifier (AC) can be run by setting `--run_AC` when launching PrepareAA

On its own, AA does not automatically produce a prediction of ecDNA or BFB status. It provides the files though that can be used to make that determination.
For more details about deciphering the AA outputs on your own, please see the [relevant section of the AA README](https://github.com/virajbdeshpande/AmpliconArchitect#outputs).
#

### Visualizing the output
AA will generate a visualization of the amplicon with edges/CN on its own. The coloring of the edges represents the [orientation of the read pairs](https://github.com/virajbdeshpande/AmpliconArchitect#4-the-sv-view-out_ampliconidpngpdf).

One common task though is, "*I have identified a likely ecDNA in my sample using AA, and would like to visualize it as a cycle*." 

We have developed a tool called **[CycleViz](https://github.com/jluebeck/CycleViz)**, which can produce a circular or linear 
plot of a path or cycle contained in an a AA cycles file. Please note the instructions for running CycleViz in the README, 
as there is a script to reformat the cycles file from AA prior to running CycleViz.
#

### Worked example
Here are some commands for running AA. We'll assume you're starting with a coordinate-sorted BAM file.

- **Run some "preflight checks"**:

```
echo $AA_DATA_REPO
echo $AA_SRC
ls $HOME/mosek/
```
None of the above should print an empty string or "no such file" error.

- **Launch AmpliconSuite-pipeline**:

In this specific case, we'll assume you don't have a CNV bed yet, and we'll assume you've installed CNVKit. Please do see the [AmpliconSuite-pipeline README](https://github.com/jluebeck/PrepareAA) though to check which flags you need to set in your case.
```
PrepareAA.py -s sample_name  -t number_of_threads --cnvkit_dir /path/to/cnvkit.py --sorted_bam sample.bam --run_AA --run_AC
```

You can still do this even if you already have CNV calls from your own caller, or if you only have the fastq files. Just check the README for the commands to use.

- **Classifying results from multiple runs**:

We can run a classification of multiple AA outputs to determine if ecDNA or other focal amps are present. You will have to install [AmpliconClassifier](https://github.com/jluebeck/AmpliconClassifier). 
```bash
# first make the .input file from multiple runs
make_input.sh /path/to/your/AA_outputs example_collection

# this will create a file called "example_collection.input" 
# next run AC
amplicon_classifier.py --ref GRCh38 --input example_collection.input
``` 

- **Creating a circular visualization of the ecDNA**:

Suppose you find that `amplicon1` of your sample `GBM39` represents an ecDNA+ amplicon, and that in the AA cycles file, the most dominant cyclic path is `Cycle1`. Perhaps you would like to visualize this cycle.

1. Install [CycleViz](https://github.com/jluebeck/CycleViz).
2. Convert your AA cycles file into the correct format 
    
    ```$CV_SRC/convert_cycles_file.py -c GBM39_amplicon1_cycles.txt -g GBM39_amplicon1_graph.txt```
3. Run CycleViz (or LinearViz).
```
# note that we use the converted cycles file
$CV_SRC/CycleViz.py --cycles_file GBM39_amplicon1_BPG_converted_cycles.txt --cycle 1 -g GBM39_amplicon1_graph.txt --ref hg19 --label_segs --gene_subset_file Bushman --gene_fontsize 9
```

At the moment, we do not support adding additional tracks of data into the plot (e.g. RNA-seq, ATAC-seq, etc.), but that is coming soon.
#

### FAQ
- **The AA_CNV_SEEDS.bed file was empty, what's wrong?**
    - Likely nothing. If no seed regions are detected, the sample likely has no candidate focal amplifications.


- **Can I use AA with whole-exome sequencing, ATAC-seq, or RNA-sequencing data?**
    - AA will fundamentally not work with these data modalities. We only support paired-end WGS data with AA.
    

- **Can I use AA on Circle-Seq data?**
    - AA has been used on Circle-Seq data with only modest success. AA is not designed for use with targeted sequencing data, and as a result it may crash or produce unreliable amplicon calls. Please proceed very cautiously if deploying AA/interpreting AA outputs on Circle-Seq data.


- **What coverage is needed for AA?**
    - Because the CN of focal amplifications is higher than the background reference, a BAM with 10x coverage will effectively have 50x coverage in a region with CN 10 (assuming 10x coverage for CN=2). Thus, even very low coverage BAM files can be used with AA.
  

- **Do I need to use downsampling? Will I detect more SVs with less downsampling?**
  - The threshold for amplified SV detection used by AA scales with effective coverage after downsampling, so lower effective coverage (e.g. downsample 10) will have a lower threshold than higher effective coverage (e.g. downsample 40). For low copy-number focal amplifications (CN < 10), higher effective coverage may perform better
(consider setting a downsample parameter of 30 or 40).  
    

- **AA has been running for more than 72 hours. What's wrong?**
    - Please ensure that you selected your CNV seeds appropriately. If you use a CN cutoff any lower than 4.5, and size < 10 kbp,
     there can be a great many false positive focal amplifications. Secondly, and very importantly, please ensure you used AmpliconSuite-pipeline to select the CNV seeds appropriately (as opposed to running `AmpliconArchitect.py` on raw, unfiltered copy number data). 
    - If AA is running for > 1 week, please check that you are using the latest version of the data repo maintained by jluebeck, and consider changing `--cnsize_min 100000 --cngain 5`, and moving up from there with your cutoffs.
     

- **Should copy-number calls provided to AA be corrected for purity and ploidy?**
    - AA's internal analysis of copy number is a coverage-based method that establishes a baseline coverage throughout the genome. 
  Copy number at a given location is also coverage based and reported relative to this baseline coverage. 
  AA itself does not consider purity or ploidy when computing its relative copy number estimates. 
  The goal is not to determine an absolutely correct copy number, but to identify focally amplified regions relative to the rest of the genome.
    - Adding corrections for purity or ploidy to the copy numbers used for seed determination may cause entire chromosome arms to appear above the copy-number cutoff for a focal amplificaiton. We recommend users set a higher CN-cutoff (e.g. `--cngain 8`) if using these kinds of corrected calls.
    - We also suggest that analyzing samples with purity below 30% should be avoided where possible. Only very-strongly amplified focal amplifications will be found. If corrected for purity, any distortions to copy number signal may also become amplified, creating many false-positive seeds.
  

- **In the amplicon visualizations, what are the different edge colors?**
    - They represent the orientation of the breakpoint. Please see the [relevant section of the AA README](https://github.com/virajbdeshpande/AmpliconArchitect#4-the-sv-view-out_ampliconidpngpdf).
  
  
- **What if I use really low CN cutoffs and really small minimum sizes so that I don't miss any ecDNA?**
    - AA will likely stall. A lower CN threshold is also not better, as in that case it may pick up many more regions which represent non-ecDNA events like segmental tandem duplications. When the CN threshold is lowered, the seed sizes also tend to expand greatly - as they may reflect karytoypic events, not focal amplifications. Giving enormous seed regions to AA, (e.g. > 10 Mbp) is strongly not recommended. 
    

- **How do I run AA on a sample with a viral genome(s) to check for integration/viral genome structure/viral ecDNA?**
  1. If needed, first determine which viral strain(s) are present (e.g. ViFi or [FastViFi](https://github.com/sara-javadzadeh/FastViFi), or other methods).
  2. Align your reads to a human reference fasta file with the relevant viral genomes included.
  3. We provide an oncoviral version of the AA data repo here: https://datasets.genepattern.org/?prefix=data/module_support_files/AmpliconArchitect/. `GRCh38_viral` should the argument given to `--ref`. If using a viral genome not present in the AA data repo, modify the AA data repo fasta file for the corresponding human build to include the viral strain(s) as entries. Re-index fasta file. Recommend copying original data repo to new location to modify, then updating `$AA_DATA_REPO` with the modified version.
  4. The virus will be added to the seed regions if present and amplified. However, if using a viral genome not present in the AA data repo, before launching AA, add the viral genome to your `AA_CNV_SEEDS.bed` file.


- **How do I tell if an amplification is due to ecDNA or segmental tandem duplication?**
    - For low CN < 4.5, it's not really possible to tell. However, keep the following in mind - when segmental tandem duplications occur, the same breakpoints must be reused every time it is repeated. When a "cyclic" structure accumulates
    in very high CN, this would involve the exact reuse of the same breakpoints many, many times in the case of segmental tandem dups. The simpler hypothesis as CN increases, is that it is mediated through an extrachromosomal DNA mechanism. 
   
 
- **Can AA determine the difference between HSR and ecDNA? Can it find integration points?**
    - From our observations, ecDNA maintains its structure when it integrates into the genome (Turner 2017, *Nature*, Deshpande 2019, *Nat. Comms.*, Luebeck 2020 *Nat. Comms.*). Unfortunately without some sort of imaging data (FISH) or long-range sequencing (Bionano, PacBio, Nanopore), it is not possible to reliably make that determination from AA.


- **There's a region I want AA to examine, but it didn't appear in the CNV seeds, what do I do?**
    - You can force AA to run on a region by creating a BED file with that region specifically included, and then invoking AA directly (do not use AmpliconSuite-pipeline or `amplified_intervals.py`). This can also be used to standardize the same AA run on multiple different samples.
 

- **In the cycles file, what do the paths that begin/end with '0+' mean?** 
    - These indicate that the path is non-cylic, and proceeds or is preceeded by the next reference coordinate.


- **Which HPV16 genome is included with GRCh38_viral?**
    - AY686584.1 (https://www.ncbi.nlm.nih.gov/nuccore/AY686584.1). The version used in the data repo (which we called hpv16ref_1) is slightly older than the current build hosted by NCBI, and differs by approximately 9 point mutations throughout the viral genome. 

#
 
