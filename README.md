# Genome annotation with Maker

<p>This repository accompanies the article Wattad et al., "Roadmap and Considerations for Genome Editing in a Non-model Organism: Genetic
Variations and Off-target Profiling", Genomics Proteomics & Bioinformatics.</p>

<p>Code and documentation for genome annotation with Maker will be uploaded here soon.</p>

## Citation

## Prerequisites

## Usage
We need to tell MAKER all the details about how we want the annotation process to proceed.
Because there can be many variables and options involved in annotation, command line options would be too numerous and cumbersome.
Instead MAKER uses a set of configuration files which guide each run.
You can create a set of generic configuration files in the current working directory by typing the following

```
maker -CTL
```

This creates three files (type ls -1 to see).


maker_exe.ctl - contains the path information for the underlying executables.

maker_bopts.ctl - contains filtering statistics for BLAST and Exonerate

maker_opts.ctl - contains all other information for MAKER, including the location of the input genome file.

## 1. RepeatMasker
Many of the steps used by MAKER can be computationally demanding, we therefore initially perform a "clean" MAKER run, which includes only RepeatMasker.
We need to build new maker configuration files and populate the appropriate values in maker_opts.ctl

`model_org=Crustacea`

We then run MAKER:
```
maker \
	-g 00.Files/Genome.fasta \
	-base "RepeatMasker" \
	01.RepeatMasker/maker_opts.ctl \
	01.RepeatMasker/maker_bopts.ctl \
	01.RepeatMasker/maker_exe.ctl
```

Once finished, we merge all the gff files generated by MAKER and extract the RepeatMasker information
```
gff3_merge \
	-n \
	-s \
	-d 01.RepeatMasker/RepeatMasker.maker.output/RepeatMasker_master_datastore_index.log \
	> 01.RepeatMasker/RepeatMasker.maker.output/RepeatMasker_model_all.gff
	
awk '{ if ($0 == "##gff-version 3" || $2 == "repeatmasker") print $0 }' 01.RepeatMasker/RepeatMasker.maker.output/RepeatMasker_model_all.gff \
	> 01.RepeatMasker/RepeatMasker.maker.output/RepeatMasker_model_repeatmasker.gff
```

## 2. Maker Round 1
We now configure MAKER to generate annotations directly from EST and protein data, which will then be used for training ab initio gene predictors. We again build new maker configuration files and populate the appropriate values in maker_opts.ctl

```
est=00.Files/macrobrachium_rosenbergii_assembly.fasta,00.Files/macrobrachium_rosenbergii_NCBI_mRNA.fasta
protein=00.Files/Crustacean_NCBI_proteins.fasta
rm_gff=01.RepeatMasker/RepeatMasker.maker.output/RepeatMasker_model_repeatmasker.gff
est2genome=1
protein2genome=1
```

We then run MAKER:
```
maker \
	-g 00.Files/Genome.fasta \
	-base "Maker_Round1" \
	02.Maker_Round1/maker_opts.ctl \
	02.Maker_Round1/maker_bopts.ctl \
	02.Maker_Round1/maker_exe.ctl
```

Once finished, we merge all the fasta and gff files generated by MAKER, separate into types and generate initial statistics
```
fasta_merge \
	-o Maker_Round1 \
	-d 02.Maker_Round1/Maker_Round1.maker.output/Maker_Round1_master_datastore_index.log

gff3_merge \
	-n \
	-s \
	-d 02.Maker_Round1/Maker_Round1.maker.output/Maker_Round1_master_datastore_index.log \
	> 02.Maker_Round1/Maker_Round1.maker.output/Maker_Round1_model_all.gff

less 02.Maker_Round1/Maker_Round1.maker.output/Maker_Round1_model_all.gff | \
	awk '$3=="mRNA"' | \
	grep "mRNA-1" | \
	awk '{print $5-$4}' \
	> 02.Maker_Round1/Maker_Round1.maker.output/00.Maker_Round1_mRNA_stats.log

python \
	00.Scripts/mRNA_stats.py \
	02.Maker_Round1/Maker_Round1.maker.output/00.Maker_Round1_mRNA_stats.log
	
perl 00.Scripts/AED_cdf_generator.pl \
	-b 0.025 \
	02.Maker_Round1/Maker_Round1.maker.output/Maker_Round1_model_all.gff \
	> 02.Maker_Round1/Maker_Round1.maker.output/00.Maker_Round1_AED_dist.log
```

In addition, to train SNAP, we need to convert the GFF3 gene models to ZFF format.
```
maker2zff \
	-x 0.25 \
	-l 50 \
	02.Maker_Round1/Maker_Round1.maker.output/Maker_Round1_model_all.gff
rename genome Maker_Round1_zff_length50_aed0.25 genome.*
```
