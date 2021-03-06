# Introduction 

After numerous requests we decided to open-source our implementation of whole genome analysis pipeline in order to share some of the best practices of running bioinformatics workflows on AWS.

Current pipeline run Best Practices GATK workflow on Illumina paired-end data and is designed to work with hg19 reference. Pipeline uses [Cirrus](http://www.clusterk.com/products/cirrus/) for task scheduler and S3 for storing intermediate files. Cirrus users can use the pipeline as is, or use it a reference to build their own workflows.

It is also possible to run the pipeline with other schedulers (such as **SGE/StarCluster**) and other filesystems (e.g. shared NFS), however it will require replacing **ksub** with appropriate call (e.g. qsub) and nmodifying **SWE** wrapper to work with your specific file system.

We present this pipeline for educational purposes only, and discourage it's use in testing or diagnostics without proper testing.

This document represents a rather incomplete description of the pipeline, and I will be gradually updating it as time permits.


## GATK Best Practices Pipeline

Currenly we expect paired end FASTQ as input and include following steps:

1. Sequence Alignment with bwa mem
2. Alignment sorting
3. Base Quality Recalibration
4. Indel realignment to refine local alignments
5. Variant calling (Unified Genotyper for GATK Lite and HaplotpeCaller othwerise)
6. Variant score recalibration
7. Publishing to S3


By default pipeline required Cirrus command line tools to be installed and submits tasks to default queue. 

Please edit general_settings.sh to point to your S3 bucket, make sure that your input files and hg19 reference is available on S3, and start sample exome workflow:

```bash
bash-3.2$ kauth --server my.cirrus.scheduler.com --login login --password passwd
bash-3.2$ bash gatk_pipeline.sh settings/exome-30x.settings 2>/dev/null
Will be using chr22 for base quality recalibration.
OK, job 343 is in batch mode with batch size 13
Processing paired[s3://gapp-east/sample/gcat_set_025_1.fastq.gz,s3://gapp-east/sample/gcat_set_025_2.fastq.gz], will split it in 2 chunks
Processing chromosome chr22
...
Processing chromosome chrM
Commiting all tasks to job 343
OK, 198 tasks from job 343 were sent to server
OK, commit of 198 tasks succeeded
bash-3.2$ 
``` 

After completion results will be available in S3, e.g.:

```bash
bash-3.2$ es3 ls s3://gapp-east/sample/GCAT_150X_Exome.UG/
1424210533	15361199	s3://gapp-east/sample/GCAT_150X_Exome.UG/recalibrated.filtered.vcf.gz
```

### Performance

When run at scale (>3 genomes at once)  using Cirrus with resource prediction enbled, AWS spot instances, default automatic scaling and using S3 for storing intermediate files we usually observe runtimes <3 hours per whole human genome (40x) and compute cost rangind from $3-$5 based on whether UnifiedGenotyper or HaplotypeCaller is used, and scales to 100-1000 of whole genomes processed at the same time.

Using other schedulers will probably yeld worse but comparable results, especially if there are performance bottlenecks such as shared NFS server.

### Accuracy 

We've run sample Illumina datasets from [BioPlantet GCAT test](http://www.bioplanet.com/gcat) that compares variants with Genome in a bottle (GiaB) reference. VCF files files are available below, let us know if your pipeline performs better!



1. [Illumina-paired-end-100bp-30x HaplotypeCaller](http://gapp-east.s3.amazonaws.com/sample/GCAT_30X_Exome.HC/recalibrated.filtered.vcf.gz)
1. [Illumina-paired-end-100bp-30x UnifiedGenotyper](http://gapp-east.s3.amazonaws.com/sample/GCAT_30X_Exome.UG/recalibrated.filtered.vcf.gz)
1. [Illumina-paired-end-100bp-150x HaplotypeCaller](http://gapp-east.s3.amazonaws.com/sample/GCAT_150X_Exome.HC/recalibrated.filtered.vcf.gz)
1. [Illumina-paired-end-100bp-150x UnifiedGenotyper](http://gapp-east.s3.amazonaws.com/sample/GCAT_150X_Exome.UG/recalibrated.filtered.vcf.gz)

 

# Detailed explanation

Entire worklow is submitted to scheduler as a directed acyclinc graph (DAG) of tasks with **ksub**. 
gatk_pipeline.sh is used to construct and sumbit task graph for execution. We briefly cover key sections of gatk_pipeline.sh below:


## Split input files

For each input file compute number of splits and submit split task to the queue for each input pair.
Split task returns task id, and after completion will emit a number of FASTQ files, that can be referred via SWE as $split_job_id:1.fastq.gz, ..., $split_job_id:N.fastq.gz


```bash
for input in $INPUT_FASTQ
do
	
    ...

	splits=$[$file1_size/$input_split_size+1]

	split_job_id=$(ksub \
						 -v splits=$splits \
						 -v input1=$file1 \
						 -v input2=$file2 \
						 --wrap="bash split_fastq/split.sh")
	...
done 

``` 


## Run BWA mem

For each individual split (e.g. 3.fastq.gz) we run alignment with BWA mem, and emit one alignment file per chromosome. (e.g. chr1.bam,...,chrX.bam).
BWA tasks depends on corresponding on corresponding split task:

```bash
		for split in $(seq 1 $splits)
		do
			# align.sh: accepts input split file, produces alignment, split by chromosome
			align_job_id=$(ksub \
							 -v sample_id=SAMPLE \
							 -v interleaved=$split_job_id:$split.fastq.gz \
							 -d $split_job_id \
							 --wrap="bash align/align.sh")
			align_job_ids="$align_job_ids $align_job_id"
		done

```

## Combine individual alignments

Alignment from multiple alignment tasks are combined together in one file per chromosome with samtools (e.g. chr5.bam):

```bash

for chr in $CHROMOSOMES
do
	
	#create comma separated list of alignment jobs
	align_job_list=$(echo $align_job_ids |tr " " ",")

	#create list of input alignment files for current chromosome
	input_array=""
	for align_job in $align_job_ids
	do
		input_array="$input_array $align_job:$chr.bam"
	done

	#submit combine jobs, and pass list of alignent files as input, and all alignment jobs as prerequsite
	#combine.sh: accepts a list of aligned bam files, produced combined file for a given chromsome
	# output: $combine_job_id:$chr.bam


	combine_job_id=$(ksub \
							-v chr=$chr \
							-v input="$input_array" \
							-d $align_job_list \
							--wrap="bash combine/combine.sh" )
	...
done

```

## Run Base quality recalibration

GATK BQSR is usually run on one the chromosomes (chr22):

```bash
for chr in $CHROMOSOMES
do
	...
	if [ "$chr" == "$BQSR_CHR" ]
	then
		#bqsr.sh: runs Base Quality recalibration on chr22
		# output is $bqsr_job_id:bqsr.grp
		bqsr_job_id=$(ksub \
						     -v chr=$chr \
						   	 -v input=$combine_job_id:$chr.bam \
						     -d $combine_job_id \
						     --wrap="bash bqsr/bqsr.sh")
	fi
	...
	
```

## Split chromosome into sub-regions

This step first scans the entire chromosome to find 1kb regions that have no chance for containing variants - no repeats, no indels and almost all bases matching reference. Chromosome is then split into N subregions using some of these safe split points, in order to make sure that no short variants (<500bp) are be affected by the splits.


```bash
for chr in $CHROMOSOMES
do
	...
 
 	gatk_splits=$[$chr_size/$chr_split_size+1]
 	
	#chr_split.sh: finds genomic locations where it is safe to split a chromosomes
	#               returns list of bam files:  1.bam, 2.bam, ... , N.bam
	
	chr_split_id=$(ksub  \
						  -v input=$combine_job_id:$chr.bam \
						  -v splits=$gatk_splits \
						  -v chr=$chr \
						  -d $combine_job_id \
						  --wrap="bash split_chr/split_chr.sh")


	...
done
```


## Run variant calling

For each split we run variant calling that includes:
1. Picard Deduplication
2. Local indel realignment
3. Application of Base Quality Recalibration data
4. Variant calling with HaplotypeCaller or UnifiedGenotyper if GATK-Lite is provided.

When UG is used, we also do additional annotation step to add MQ0 score to alignment which significantly improves quality of Variant recalibration in the next step.

```bash
for chr in $CHROMOSOMES
do
	...

	gatk_job_ids=""
	for split_id in $(seq 1 $gatk_splits)
	do

		#gatk.sh runs gatk on a sub-interval and applies BQSR
		#output: $gatk_job_id:raw.vcf
		#submit GATKs with lower priority to let chr_splits finish faster
		gatk_job_id=$(ksub \
							-v input=$chr_split_id:$split_id.bam \
							-v interval=$chr_split_id:$split_id.interval \
							-v bqsr=$bqsr_job_id:bqsr.grp \							-d $chr_split_id,$bqsr_job_id \
							--wrap="bash gatk/gatk.sh")
							
		gatk_job_ids="$gatk_job_ids $gatk_job_id"
		
	done	
		
	...
done
```



## Combine partial VCFs into one

Multiple partial VCF files are combined into one.

```bash

input_array=""
for combine_vcf_job in $combine_vcf_job_ids
do
    input_array="$input_array $combine_vcf_job:raw.vcf"
done

combine_vcf_job_id=$(ksub   \
			    -v input="$input_array" \
			    -d $combine_vcf_job_list \
			   --wrap="bash combine_vcf/combine_vcf.sh" )

```

## Run VQSR

Variant Quality Score Recalibration is performed separately for Indels and SNPS, variants are merged into final VCF file.

```bash
	#run variant quality recalibration
	# output: $vqsr_job_id:recalibrated.filtered.vcf.gz


vqsr_job_id=$(ksub \
		    -v input=$combine_vcf_job_id:raw.vcf \
		    -v ANALYSIS=$ANALYSIS \
		    -d $combine_vcf_job_id \
		    --wrap="bash vqsr/vqsr.sh")

```

## Publish results to S3

Final VCF file is saved on S3 using sample name as directory prefix. 

```bash
	# save results to S3 and make them publicly accessible over HTTP
publish_job_id=$(ksub \
		    -v input=$vqsr_job_id:recalibrated.filtered.vcf.gz \
		    -v path="$SAMPLE_DATA/$NAME" \
		    -d $vqsr_job_id \
		    --wrap="bash publish/publish.sh")
		    
```

# SWE

to be described soon

# Cirrus specific command lines
to be described soon


## Included 3rd party software
Coming soon.


