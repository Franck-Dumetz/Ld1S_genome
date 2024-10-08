# Ld1S_genome
Making genome assembly of Ld1S2D and further comparison with BPK282A2 (TritrypDB v63)

* mapping CCS reads to LdBPK282A2 using minimap2

minimap2 -ax map-hifi -t 2 <PATH-TO-REF-GENOME>/TriTrypDB-63_LdonovaniBPK282A1_Genome.fasta <PATH-TO_CCS_READS>/PACBIO_DATA/EDS10_20230707_S64411e_PL100299966A-1_A01_bc2088-bc2088.ccs.fastq.gz > Ld1S_282aligned.sam
samtools view -bhF 2308 Ld1S_282aligned.sam | samtools sort -o Ld1S_282aligned.bam
samtools index Ld1S_282aligned.bam

* Genome assembly strategies
   * Using Flye

export PATH=/usr/local/packages/flye-2.9/bin:$PATH  
/usr/local/packages/flye-2.9/bin/flye --pacbio-hifi /PACBIO_DATA/EDS10_20230707_S64411e_PL100299966A-1_A01_bc2088-bc2088.ccs.fastq.gz --genome-size 33m --out-dir /local/projects-t3/SerreDLab-3/fdumetz/Leishmania/Ld1S2D_genome/Flye -t 16

   * Using Canu

/usr/local/packages/canu-2.1.1/bin/canu -assemble -pacbio-hifi /PACBIO_DATA/EDS10_20230707_S64411e_PL100299966A-1_A01_bc2088-bc2088.ccs.fastq.gz

----------------------------------------------------------------------------------------------------------------------------------------------------------------------

* Checking genome assembly quality compared to LdBPK282A2
  * using quast

/usr/local/packages/quast-5.2.0/quast.py -r /local/projects-t3/SerreDLab-3/fdumetz/Genomes/LdBPK282A2/TriTrypDB-63_LdonovaniBPK282A1_Genome.fasta -g /local/projects-t3/SerreDLab-3/fdumetz/Genomes/LdBPK282A2/TriTrypDB-63_LdonovaniBPK282A1.gff -o quast_282vsLd1S --min-contig 1000 -t 4 --gene-finding Ld1S_assembly_final.fasta

  * using Mummer

/usr/local/packages/mummer-3.23/nucmer -p Ld1Svs282_WG_nucmer --mum <PATH-TO-REF-GENOME>/TriTrypDB-63_LdonovaniBPK282A1_Genome.fasta <PATH-TO-QUERY_GENOME>/Ld1S_assembly_final.fasta

/usr/local/packages/mummer-3.23/mummerplot --png --filter --color --layout --prefix=Ld1Svs282_WG_nucmer Ld1Svs282_WG_nucmer.delta -R <PATH-TO-REF-GENOME>/TriTrypDB-63_LdonovaniBPK282A1_Genome.fasta -Q <PATH-TO-QUERY_GENOME>/Ld1S_assembly_final.fasta

* Rearrangement

/usr/local/packages/mummer-3.23/show-diff Ld1Svs282_WG_nucmer.delta > Ld1Svs282_WG_nucmer.rearrangement

* Repeats

/usr/local/packages/mummer-3.23/nucmer --maxmatch --nosimplify --prefix=Ld1Svs282_WG_nucmermax /local/projects-t3/SerreDLab-3/fdumetz/Genomes/LdBPK282A2/TriTrypDB-63_LdonovaniBPK282A1_Genome.fasta /local/projects-t3/SerreDLab-3/fdumetz/Leishmania/Ld1S_genome/Flye_scaffold/Ld1S_assembly_final.fasta

/usr/local/packages/mummer-3.23/show-coords -r Ld1Svs282_WG_nucmermax.delta > Ld1Svs282_WG_nucmermax.coords

* SNP density

  * MUMmer SNP comparition

Using nucmer to identify SNP and position

/usr/local/packages/mummer-3.23/nucmer -mum <PATH-TO-REF-GENOME>/TriTrypDB-63_LdonovaniBPK282A1_Genome.fasta <PATH-TO-QUERY_GENOME>/Ld1S_assembly_final.fasta

/usr/local/packages/mummer-3.23/delta-filter -r -q Ld1Svs282_WG_nucmer.delta > Ld1Svs282_WG_nucmer.filter

/usr/local/packages/mummer-3.23/show-snps -Clr Ld1Svs282_WG_nucmer.filter > Ld1Svs282_WG_nucmer.snps

* Filtering Ld1Svs282_WG_nucmer.snps to extract only SNPs (no insertion, no deletion)

tail -n +6 Ld1Svs282_WG_nucmer.snps | awk '{if($2!=".") print}' | awk '{if($3!=".") print $1"\t"$14}' > SNP_density.tsv

* Importing the dataset into R to plot the density for all chromosomes

#!/usr/bin/env Rscript

library(ggplot2)

colnames(SNP_density) <- c("Position", "Chr")
SNP_density$Position <- as.numeric(SNP_density$Position)

head(SNP_density)

ggplot(aes(Position, group = Chr), data=SNP_density) +
    geom_histogram(binwidth = 5000) + 
    #facet_wrap(~Chr, scales = "free") +
    scale_x_continuous(labels = function(x) format(x, scientific = TRUE))

* Local gene copy number variation using base converage

  * base coverage using mpileup

samtools mpileup /local/projects-t3/SerreDLab-3/fdumetz/Genomes/LdBPK282A2/TriTrypDB-63_LdonovaniBPK282A1_Genome.fasta /local/projects-t3/SerreDLab-3/fdumetz/Leishmania/Ld1S_genome/CCS_282_aligned/Ld1S2d_BPKaligned.bam

  * extracting data from the mpileup file

awk '{print $1"\t"$2"\t"$4}' Ld1Svs282_mpileup.txt > Ld1Svs282_PosCov.tsv

grep Ld01_v1s1 Ld1Svs282_PosCov.tsv > Ld1_cov.tsv

  * Plotting coverage chromosome per chromosome with R

library(ggplot2)
library(dplyr)

colnames(Ld36_cov) <- c("Chr", "Position", "Coverage")
Ld1_cov$Coverage <- as.numeric(Ld1_cov$Coverage)

Ld1_cov$window <- Ld1_cov$Position/5000
Ld1_cov$window <- as.integer(Ld1_cov$window)

Ld1_cov_sum <- Ld1_cov %>% 
  group_by(window) %>%
  summarise(sum_cov = sum(Coverage))

head(Ld1_cov)

ggplot(Ld1_cov_sum, aes(x=window, y=sum_cov)) +
  geom_line() +
  theme_classic() +
  geom_hline(yintercept = mean(Ld36_cov_sum$sum_cov), color="red")

----------------------------------------------------------------------------------------------------------------------------------------------------------------------

* tRNA destection using tRNA scan

/usr/local/packages/trnascan-se-2.0.3/bin/eufindtRNA -r <PATH>/Ld1S_assembly_final.fasta > Ld1S_tRNA_strict.csv 

----------------------------------------------------------------------------------------------------------------------------------------------------------------------

* snoRNA detection using snoReport2.0

export SNOREPORTMODELS="/usr/local/packages/snoreport-2.0/models"   #### make  sure the "models" folder is readable or it won't work

/usr/local/packages/snoreport-2.0/snoreport_2 -i /local/projects-t3/SerreDLab-3/fdumetz/Leishmania/Ld1S_genome/Flye_scaffold/Ld1S_assembly_final.fasta -CD -HACA -o Ld1S_snoRNA --PS

----------------------------------------------------------------------------------------------------------------------------------------------------------------------

* Using BUSCO

export PATH="/usr/local/packages/augustus-3.4.0/bin:$PATH"

export PATH="/usr/local/packages/augustus-3.4.0/scripts:$PATH"

export AUGUSTUS_CONFIG_PATH="/usr/local/packages/augustus-3.4.0/configs/"

export PATH="/usr/local/packages/metaeuk-6-a5d39d9/bin:$PATH"

busco -m genome -i <PATH-TO-QUERY_GENOME>/Ld1S_assembly_final.fasta --auto-lineage-euk --long -o busco  -f

----------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Using the ONT reads to refine trasncripts annotation

* Basecalling fast5 files (all fast5 files were basecalled using Guppy on an Nvidia GPU)

/usr/local/packages/guppy-6.4.2_gpu/bin/guppy_basecaller -x "cuda:0" --input_path "$fast5_dir" --save_path "$output_dir" --config rna_r9.4.1_70bps_hac.cfg --min_qscore 7 --records_per_fastq 10000000 --gpu_runners_per_device 8 --num_callers 1 (--trim-strategy none)

* read alignment using minimap.sh script

fastq_dir=/local/projects-t3/SerreDLab-3/fdumetz/Leishmania/Ld_ONT/Annotation/282_aligned

fastq_file1=/local/projects-t3/SerreDLab-3/fdumetz/Leishmania/Ld_ONT/Annotation/Ld_3ONT.fastq.gz

sam_file1=Ld_3ONT_282.sam

ref_file=/local/projects-t3/SerreDLab-3/fdumetz/Leishmania/Ld1S_genome/Flye_scaffold/Ld1S_assembly_final.fasta

minimap2 -ax map-ont -t 2 "$ref_file" "$fastq_file1" > "$sam_file1"

samtools view -bhF 2308 Ld_3ONT_282.sam | samtools sort -o Ld_3ONT_282.bam

samtools index Ld_3ONT_282.bam
