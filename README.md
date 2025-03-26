# Helixer-and-EDTA
A more in depth look at  repeat masking and gene preditions

### Training helixer on extrinsic data
 For organisms that are far from the prebuilt helixer models it is sometimes required to also train the final layers of the closest model with the genome RNA alignments.
 The first step in doing this requires the genome assembly that is being aligned to the RNA reads is converted to a format that is readable by the LLM this is done with the following command.

```
singularity exec --nv --env 'RUST_BACKTRACE=full' /lustre/isaac/scratch/jtorre28/singularity_containers/helixer-docker_helixer_v0.3.3_cuda_11.8.0-cudnn8.sif import2geenuff.py \
--fasta $fasta \
--gff3 $gff \
--db-path ${species}.sqlite3 \
--log-file ${path}/${species}.log \
--species $species \
```

> NOTE: Througout this guide their will be predefined files and variables that are best to have already ready and defined before starting.
> They are:
> ```
> path='/lustre/isaac/scratch/jtorre28/pandora-model'
> fasta="/lustre/isaac/scratch/madler5/blue_pandora_seq/Genome_Assembly/hifiasm/hifiasm/pacbio-ont-sana-hic-1.hic.p_ctg.fa"
> gff="/lustre/isaac/scratch/madler5/blue_pandora_seq/Genome_Assembly/Genome_Annotation/1_helixer_docker/pandora-norrna.gff"
> species="pandora-fungus"
> prediction="/lustre/isaac/scratch/jtorre28/helixer-pand-half/predictions.h5"
> model="/nfs/home/jtorre28/.local/share/Helixer/models/fungi/fungi_v0.3_a_0100.h5"
> bam='/lustre/isaac/scratch/madler5/blue_pandora_seq/RNAseq_Processing/2_Trimmomatic/posttrim_paired/mapped-rna.bam'
> ```

Once this initial convertion is finished it still needs to be converted to the final h5 format

```
singularity exec --nv --env 'RUST_BACKTRACE=full' /lustre/isaac/scratch/jtorre28/singularity_containers/helixer-docker_helixer_v0.3.3_cuda_11.8.0-cudnn8.sif \
geenuff2h5.py --h5-output-path ${species}.h5 \
--input-db-path ${species}.sqlite3
```

Now that the genome is in a readable format the genome can be run thorugh helixer's initial prediction then filtered for its most confident predictions for later

```
singularity exec --nv --env 'RUST_BACKTRACE=full' /lustre/isaac/scratch/jtorre28/singularity_containers/helixer-docker_helixer_v0.3.3_cuda_11.8.0-cudnn8.sif \
python3 /lustre/isaac/scratch/jtorre28/pandora-model/filter-to-most-certain.py --write-by 6415200 \
--h5-to-filter ${species}.h5 --predictions ${prediction} \
--keep-fraction 0.2 --output-file filtered.${species}.h5
```
Assuming the RNA reads have already been mapped and sorted to the genome the reulting bam file can be uploaded to the h5 file in the following command

```
singularity exec --nv --env 'RUST_BACKTRACE=full' /lustre/isaac/scratch/jtorre28/singularity_containers/helixer-docker_helixer_v0.3.3_cuda_11.8.0-cudnn8.sif \
python3 /home/helixer_user/Helixer/helixer/evaluation/add_ngs_coverage.py \
-s ${species} --unstranded \
--bam ${bam} --h5-data filtered.${species}.h5 \
--threads 45
```

The filtered h5 prediction file from earlier is then split into 2 seperate files for self training

```
singularity exec --nv --env 'RUST_BACKTRACE=full' /lustre/isaac/scratch/jtorre28/singularity_containers/helixer-docker_helixer_v0.3.3_cuda_11.8.0-cudnn8.sif \
python3 /lustre/isaac/scratch/jtorre28/pandora-model/n90_train_val_split.py --write-by 6415200 \
--h5-to-split filtered.${species}.h5 --output-pfx splitter/
```

All of this information is then finally used to train/fine-tune the final layers of the appropiate pre-built model 

```
singularity exec --nv --env 'RUST_BACKTRACE=full' /lustre/isaac/scratch/jtorre28/singularity_containers/helixer-docker_helixer_v0.3.3_cuda_11.8.0-cudnn8.sif \
HybridModel.py -v --batch-size 140 --val-test-batch-size 280 \
   --class-weights "[0.7, 1.6, 1.2, 1.2]" --transition-weights "[1, 12, 3, 1, 12, 3]" \
   --predict-phase --learning-rate 0.0001 --resume-training --fine-tune \
   -l $model \
--input-coverage --coverage-norm log --data-dir splitter/ --save-model-path ${path}/tuned-${species}-model.h5
```

### EDTA pipeline

```
EDTA.pl --genome  /lustre/isaac/scratch/madler5/blue_pandora_seq/Genome_Assembly/hifiasm/hifiasm/pacbio-ont-sana-hic-1.hic.p_ctg.fa --sensitive 1 -t 48 --debug 1 --anno 1
```

> Options used:
> 
> --sensitive  : When set to "1" RepeatModeler is used to make a repeat library for TE masking
> 
> --anno 1     : When set to "1" GFF file included in output
> 
> -t           : Number of threads/CPU assigned
> 
> --debug      : When set to "1" will add debug to error output








