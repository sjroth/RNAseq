# RNAseq
Basic RNAseq Processing Pipeline using Singularity and Snakemake.

## Prerequisites
The prerequisites for this pipeline are minimal due to Singularity. All you need is to install Singularity. This package installs all prerequisites for software. [Singularity Installation](http://singularity.lbl.gov/docs-installation)

Other than singularity, there are a few requirements. The first is your directory structure. This pipeline takes advantage of a regularized directory structure. Conveniently, most of the directories are generated by Snakemake automatically so your input is minimal. You need to define a parent directory with a folder in it labeled data. Data will have one subdirectory called raw_data. As the name implies, that is where you put your fastq.gz files. NOTE: this pipeline does no QC so be sure that your files are trimmed and gzipped beforehand. Here is how the setup of your directories can look:
```
mkdir -p PARENT_DIRECTORY/data/raw_data 
cp RAW_FILES raw_data
```
After depositing raw files into the raw_data file, it's time to load other prequisites. For now, this script does not automatically generate the 3 prerequisite files necessary (perhaps something for future modifications). These files are the chromosome sizes file, the Kallisto index file, and the STAR reference folder. You should maintain these independently of the pipeline and copy them into your data directory after. Naming is important for this pipeline so be sure to name anything that is lowercase as specified. Here is how you would create these files and then copy them into your data folder:
```
samtools faidx GENOME.fa
cut -f1,2 GENOME.fa.fai > GENOME.chrom.sizes
cp GENOME.chrom.sizes PARENT_DIRECTORY/data/chrom.sizes

kallisto index -i GENOME.idx  TRANSCRIPTS.fa
cp GENOME.idx PARENT_DIRECTORY/data/kallisto.idx

mkdir STAR_GENOME
STAR  --runMode genomeGenerate --runThreadN NUM_THREADS --genomeDir STAR_GENOME --genomeFastaFiles GENOME.fa
cp -r STAR_GENOME PARENT_DIRECTORY/data/star_reference
```
Here are the [Kallisto](https://pachterlab.github.io/kallisto/manual) and [STAR](https://github.com/alexdobin/STAR/blob/master/doc/STARmanual.pdf) manuals for reference on those indices. Note that if you already have your desired files, simply omit the generation steps and copy them into the directory.

## Getting Started
Once you have satisfied the prerequisites, there are two more steps until you get going. First, alter your config file as necessary. There are only 3 fields: tag directory generation commands, kallisto commands, and whether the directory is paired end. I'll cover a few common commands.

For your tag directory generation, use the [Homer Manual](http://homer.ucsd.edu/homer/ngs/tagDir.html) to configure commands. The most common commands are -flip and -sspe. -flip is necessary for some protocols where the strands are flipped when aligned (common with some sequencing preps) and -sspe is strand-specific paired end (which is necessary if you have paired end data). Here's the config category if you need both of these:
```
tag_dir_cmds: -flip -sspe
```
It is that simple!

The Kallisto commands are a bit more complicated. Similar to above, if you need the strands to be flipped, there is a command in kallisto (--rf-stranded). If it's a strand-specific RNA-seq that doesn't need to be flipped, you need the --fr-stranded command. However, if you have single-end data, you need to flag it with --single and specify a fragment length (-l) and a standard deviation (-s). Check out the Kallisto manual linked above for more details. Here's an example for single-end enda that needs to be flipped:
```
kallisto_cmds: --rf-stranded --single -l FRAGMENT_LENGTH -s STANDARD DEVIATION
```
Please consult the Kallisto manual above since these commands are important for transcript quantification.

Finally, you need to specify whether your data is paired end or not. This is ultra-simple. Mark this True or False on the config file. This is simple. For a paired end dataset,
```
paired_end: True
```
At the present moment, you need to use RNAseq datasets that are processed in the same way.

The second part of this is generating the Singularity simg file. Use the Singularity manual for installation. It is required that you do this on a computer where you have sudo privileges and after you alter your config file. The command is simple:
```
sudo singularity build rnaseq.simg Singularity
```
Do this inside the directory and it will generate the rnaseq.simg file. This takes some time. Then, scp or cp the file to your directory of interest.
```
scp rnaseq.simg PARENT_DIRECTORY
```
Once you have the simg file, do a dry run in order to test that everything is in the right place.
```
singularity run --bind data/:/scif/data rnaseq.simg run snakemake '-n'
```
This will deposit the config file in your data folder so that you can edit it with the shell.
## Running the pipeline
It is recommended that you run this pipeline outside the Singularity environment. You also need to specify the number of cpus to use on this job. With the current setting, the maximum is 50. Here is how you run the pipeline:
```
singularity run --bind data/:/scif/data rnaseq.simg run snakemake '-j NUM_CPU'
```
