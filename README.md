# PhyloHerb
**Phylo**genomic Analysis Pipeline for **Herb**arium Specimens

This bioinformatic tutorial provides detailed guidance to process **genome skimming** data collected from herbarium specimens. The outcomes include plastid genome (plastome) assemblies, mitochondrial genome assemblies, nuclear 35S ribosomal DNAs (NTS+ETS+18S+ITS1+5.8S+ITS2+25S), alignments of gene and intergenic regions, and a species tree. Combined with the morphological and distribution data from herbarium specimens, this approach provides an unparalleled opportunity to study **taxonomy, biogeography, and macroevolution with nearly complete taxon sampling**.

We have tested this pipeline in the Barbados Cherry family Malpighiaceae, Clusiaceae, and several groups of algae. Each of these datasets contains hundreds to thousands of species and our pipeline extracts ample data to resolve both recent radiations (e.g., *Bunchosia*, Malpighiaceae >135 sp within 10 Myr) and ancient divergences (e.g., the divergence of red algea at hundreds of millions of years ago). 

## I. Prerequisites
To process large datasets (>20 sp), high performance cluster is recommended. Mac and PC can suffer from insufficient memory during the assembly, alignment, and phylogenetic reconstruction. Installation instructions for some of the following software can be found [here](/botany2021_tutorial/README.md).

### Assembly
1. [GetOrganelle v1.7.0+](https://github.com/Kinggerm/GetOrganelle)
2. [Bowtie2 v2.2.2+](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml)
3. [BLAST+](https://blast.ncbi.nlm.nih.gov/Blast.cgi?PAGE_TYPE=BlastDocs&DOC_TYPE=Download)
4. Assembly viewer: [Bandage](https://rrwick.github.io/Bandage/)
5. Optional assembly viewer: [Geneious](https://www.geneious.com/) (the complementary version is sufficient)

### Alignment
6. [Biopython](https://biopython.org/)
7. Aligner: 

	[Pasta](https://github.com/smirarab/pasta) for highly variable regions such as the ITS sequences
	
	[MAFFT](https://mafft.cbrc.jp/alignment/software/) for less variable regions or long alignments (>5 kb) that pasta may not be able to handle when the number of species is high (>500 sp)
8. Manual assembly examination: [Geneious](https://www.geneious.com/) (the licensed version are required)

### Phylogeny
9. [IQ-TREE](http://www.iqtree.org/) or [RAxML](https://cme.h-its.org/exelixis/web/software/raxml/) or [ExaML](https://cme.h-its.org/exelixis/web/software/examl/index.html)

## II. General guidelines for genome skimming data collection

**For the impatient:**

If interested in phylogeny alone, up to 384 samples (4 plates * 96 samples/plate) can be multiplexed on a single Illumina HiSeq 2500 lane for most flowering plants. Using the NovaSeq plastform can generate more complete genomes due to its larger output, but currently we cannot put more than 384 multiplexed samples due to the barcode limitation. If circularized plastid genomes are needed, >2 Gb data per species can usually get you there, which translates to ~60 samples per lane.

*IMPORTANT*: If your species have fewer-than-usual plastids per cell or exceptionally large genomes, you need to reduce the number of multiplexed species per sequencing lane. Use the following equation to calculate the expected base coverage of plastid genome:

<img src="/images/plastid_perc.png" width="600" height="100">

Minimally, you want the plastid coverage to be larger than 10X.

**FAQ**

1. DNA extraction from herbarium specimens? How?!

We have successfully extracted DNA from 200-year-old specimens. Age matters less than the preservation methods (see [this paper](https://www.frontiersin.org/articles/10.3389/fevo.2019.00439/full)). Standard commercial DNA extraction kits are frequently used to obtain DNA (e.g, Tiangen DNAsecure Plant Kit, Qiagen DNeasy Plant Mini Kit). We used a [Promega Maxwell](https://www.promega.com/products/lab-automation/maxwell-instruments/maxwell-rsc-instrument/?catNum=AS4500) instrument that can process 16 DNA samples simultaneously and extract their DNAs within an hour. This automatic approach is certainly more labour efficient, but manual extractions have more guaranteed yields for delicate precious samples.

2. Where can I find the genome sizes of my species?

In addition to searching through the literature or conduct your own flow cytometry experiments, you could also check the [Plant DNA C-value database](https://cvalues.science.kew.org/) put together by Kew.

3. NGS library preparation and multiplexing

We used the [KAPA HyperPlus Kit](https://sequencing.roche.com/en/products-solutions/products/sample-preparation/dna-reagents/library-preparation/kapa-hyperplus.html) for NGS library. Many institutes provided services for NGS library preparation with robots. We have used quarter reaction (1/4 of all reagents) for our NGS libraries, and it works just fine.

4. Where are the limits?

About 1-3% of the reads from genome skimming are from plastomes. The base coverage is roughly half for mitochondria and 2X for nuclear ribosomal regions compared to plastids. Theoretically the base coverage of plastome vary with the size of the nuclear genome and the abundance of plastids within a cell, but we found it to be relatively consistent across flowering plant species despite the dramatic difference in their genome sizes (200 Mb to 3Gb). Below is a **very rough** guidance of what you may expect from certain amount of input data.

<img src="/images/coverage.png" width="400" height="130">

## III. Assembly

We will use [GetOrganelle](https://github.com/Kinggerm/GetOrganelle) to assemble the plastome, mitochondrial genome, and ribosomal regions. It requires minimal tweak for various types of data. I highly recommend [installing it using conda](https://github.com/Kinggerm/GetOrganelle#installation--initialization) so that all its dependencies are in your environment.

### 1. Input:

Illumina FASTQ reads for each species, single-ended or pair-ended, zipped or unzipped. Do not filter the reads or trim adapters, GetOrganelle will take care of it.

### 2. How to:

After loading GetOrganelle to your environment, the basic commands for running assembly with pair end data is as follows:

```
#To assemble plant plastome
get_organelle_from_reads.py -1 <forward.fq> -2 <reverse.fq> -o <plastome_output> -R 15 -k 21,45,65,85,95,105 -F embplant_pt

#To assemble plant nuclear ribosomal RNA
get_organelle_from_reads.py -1 <forward.fq> -2 <reverse.fq> -o <nr_output> -R 10 -k 35,85,105,115 -F embplant_nr

#To assemble plant mitochondria:
get_organelle_from_reads.py -1 <forward.fq> -2 <reverse.fq> -o <mito_output> -R 50 -k 21,45,65,85,105 -P 1000000 -F embplant_mt
```

If you want to use your own reference sequences for assembly, you can provide the seed fasta file by adding `-s <reference.fas>`.

### 3. Large dataset and batch submission to cluster

If you are dealing with large number of species, running them one by one is too tedious. Here, we will submit individual assembly task to the cluster and run them simultaneously. An example bash file is provided in `/phyloherbLib/getorg.sh`. We will also use short and informative output prefix for each species. You can submit your job by typing

```
sbatch getorg.sh <forward.fq> <backward.fq> <output prefix>
```
*IMPORTANT*: Make sure you load the correct environment and provide absolute path to the input data if they are not in the current directory by modifying relavant variables in `getorg.sh`. Instructions for single-end data can also be found in `getorg.sh`.

### 4. Output

The batch submission will generate three subdirectories `chl/`, `ITS/`, and `mito/`, each containing Getorganelle output directories named after sample-specific prefixes.

### 5. Assembly visualization with Bandage


### 6. Assembly QC 

After the assemblies are completed, you can summarize the results using the `qc` function of phyloherb. For each species, it will extract the following information: the number of total input reads, the number of reads used for assembly, average base coverage, the total length of the assembly, GC%, and whether the assembly is circularized. 

```
python phyloherb.py -m qc -s sample_sheet.tsv -i <directory containing assemblies> -o <output directory>
```
This command will copy all of the assemblies under the input directory to a new directory and rename the files based on their species prefixes. In the output directory, you will also find a summary spreadsheet `assembly_sum.tsv` with the following header:
```
sp_prefix	Total_reads	Reads_in_target_region	Average_base_coverage	Length	GC%	Circularized
```

## IV. Annotation and organellar structure variarion
Annotation is not necessary if you are interested in phylogeny alone, but if you want to submit your circularized assemblies to GenBank or extract intergenic regions from your spcecies, it is a must.

The most convenient tool I have used is the web-based tool [GeSeq](https://chlorobox.mpimp-golm.mpg.de/geseq.html). I have concatenated 100 plastomes into a single fasta file and annotated them all at once on GeSeq. But if you are annotating hundreds of plastomes, command-line based tools like [PGA](https://github.com/quxiaojian/PGA) might be a better option.

## V. Ortholog identification

Phyloherb will identify the best-matching region of each gene/intergenic region in the assemblly using BLAST. We provide a build-in database of ## plastid genes from 100 angiosperm species. This database is sufficient for getting genes from species that are not too distantly related to our reference species. You can also supply your own reference in a fasta file following the instructions below. The list of our reference species is [here](/phyloherbLib/reference_sp.list). The list of the genes in the database is [here](/phyloherbLib/gene.list). 

1. Generate reference gene sequences (optional)

You can obtain your gene references from GenBank or the alignment files generated by GeSeq above. All references can be included in a single fasta file `gene_ref.fas`. The headers should begin by the gene name then followed by an underscore and a species name (which will be ignored):

```
>gene1_sp1
atcg...
>gene1_sp2
atcg...
>gene2_sp1
atcg...
>gene2_sp2
```

2. Extract orthologous gene or intergenic regions from the assembly

Assuming all of the assemblies are stored in the directory `2_assemblies/chl`, we can  extract the target gene regions using the `ortho` function of phyloherb. 
```
python phyloherb.py -m ortho -i <directory containing assemblies> -o <output directory>
```
You can choose to extract a subset of genes from a subset of the species by supplying a `-g gene_subset.txt` and `-sp species_subset.txt`. Example files can be found in [gene_subset.txt](phyloherbLib/gene_subset.txt) and [species_subset.txt](phyloherbLib/species_subset.txt).
```
python phyloherb.py -m ortho -i <directory containing assemblies> -o <output directory> -g <gene list> -sp <species list>
```
In the output directory, orthologous genes will be written to separate fasta files and the headers will be the species prefixes.

3. Intergenic regions

4. Alignment

I like to use the `--adjustdirection` function from `MAFFT` to correct reverse complimentary sequences. Then I will use `pasta` to more accurately align high variable sequences such as the intergenic regions and the ITS regions. It first generates a guidance tree, then align among closely-related species, finally merge the alignments to produce the output.

This is a potentially time consuming step so I recommend running it on the cluster using the example batch file [mafft_pasta.sh](phyloherbLib/mafft_pasta.sh).

Copy `mafft_pasta.sh` to the same directory where the gene sequences are located. Modify the file to include appropriate environmental parameters. Then the batch job can be submitted to the cluster by typing
```
sbatch mafft_pasta.sh <gene_1>
sbatch mafft_pasta.sh <gene_2>
```

5. Nuclear ribosomal and mitochondrial regions

The nuclear ribosomal data requires a slightly different curation strategy. The highly variable sequence requires more manual curation than the plastome. The nuclear ribosomal region exists as tandem repeats on multiple chromosomes.

<img src="/images/ITS.png" width="400" height="400">

Based on our experiences, NTS is not alignable even between closely related taxa. The entire rDNA region (18S+ITS1+5.8S+ITS2+25S) and some portion of ETS can be aligned at family level. 

We will try to align the entire region using MAFFT first.   

6. mitochondrial regions

For most plant groups, mitochondria are not phylogenetically informative because the genes evolve too slowly, but the intergenic regions are highly variable. Moreover, the qualities of mitochondrial genomes are usually not as good as plastomes. So we will only extract mitochondrial genes for comparative purposes. The methods is similar to plastid genes.


7. Manual curation in Geneious

At this point, it is recommended to take a initial look at your alignments. **Initial** means be prepared to complete the alignment-manual check-phylogeny cycle for at least two rounds to get publication quality data.

The purpose of the initial check is to remove obvious low-quality sequences. Do not conduct any site-based filtering yet! For example, the two sequences highlighted in red below contain too many SNPs (marked in black). They should be removed.

<img src="/images/Geneious.png" width="600" height="400">

Geneious is the best tool for this task. You can view statistics of your alignments, delete sequences, and concatenate alignments. The down side is that it is not free. Alternative automatic tools include [trimAL](http://trimal.cgenomics.org/getting_started_with_trimal_v1.2). But I highly recommend visualizing your data.


## VI. Phylogeny reconstruction

1. Concatenation

Many tools are available for concatenating alignments. I recommend the `conc` function of phyloherb or Geneious. I have applied both tools to dataset with 1000 sp x 100 genes. The `conc` function of phyloherb will also output a gene delineation file required by `PartitionFinder`.

To use the `conc` function of phyloherb, use the following commands
```
python phyloherb.py -m conc -i <directory containing assemblies> -o <output directory>
```
This command will concatenate all of the fasta sequences in the input directory with the specified suffix. Again, if you only want to use a subset of the genes or want the genes to appear in a specific order, you can supply a gene order file by adding `-g gene_subset.txt`.

2. Maximum likehood phylogeny

For an initial quick and dirty phylogeny, I recommend ExaML with unpartitioned alignment. IQTREE or RAxML generates more accurate estimations of the phylogeny and substitution paramters, but may not accomodate thousands of species with millions of sites.

3. Second round of manual alignment curation

It can be a quite satisfying experience as you browse through a well-curated alignment. To get us there, we need to conduct a second round of alignment curation and remove spurious regions arising from assembly errors or false positive BLAST hits. 

First, using a reference ExaML species tree (newick format), we will order the sequences based on their phylogenetic affinity. This can be done using the `order` function of phyloherb. If you want to additionally filter sequences based on missing data, using the optional `--missing` flag
```
python phyloherb.py -m order -t <reference.tre> -i <directory containing assemblies> -o <output directory> --missing <float number 0 to 1>
```

This will generate an ordered alignment `*.ordered.fas` and a companion tree file `*.pasta_ref.tre` for each gene. You will need this tree for the second round of pasta alignment after manual curation.

Now let's load the ordered alignments to Geneious for some fine tuning. This time we will delete blocks of problematic sequences. They usually appears as a cluster of SNPs highlighted in red below. These SNPs are not conserved in their close relatives, so they are phylogenetically uninformative autapomorphies. Regardless of the causes, we can safely delete them. 

<img src="/images/Geneious2.png" width="400" height="250">