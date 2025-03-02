# Ld1S genome assembly

BioProject SUB15103653

Table of content: <br />
[Long reads mapping with minimap2](https://github.com/Franck-Dumetz/Ld1S_genome/blob/main/README.md#mapping-ccs-reads-to-ldbpk282a2-using-minimap2) <br />
[Genome Assembly Strategies](https://github.com/Franck-Dumetz/Ld1S_genome/blob/main/README.md#genome-assembly-strategies) <br /> 
[Assembly quality assessment](https://github.com/Franck-Dumetz/Ld1S_genome/blob/main/README.md#assembly-quality-assessment) <br />

Software requirements: <br />
• busco-5.4.3 <br />
• canu-2.1.1 <br />
• flye-2.9 <br />
• guppy-6.4.2 <br />
• minimap2.1 <br />
• mummer-3.23 <br />
• quast-5.2.0 <br />
• samtools-1.20 <br />
• trnascan-se-2.0.3 <br />


## mapping CCS reads to LdBPK282A2 using minimap2
```
ref_gen=path_to_ref_genome
ccs_reads=path_to_ccs_reads
out_sam=output_sam_name
out_bam=output_bam_name

minimap2 -ax map-hifi -t 8 $ref_gen $ccs_reads > $out_sam
samtools view -bhF 2308 $out_sam | samtools sort -o $out_bam
samtools index $out_bam
```

## Genome assembly strategies
   ### Using Flye
   
```
export PATH=/usr/local/packages/flye-2.9/bin:$PATH

ccs_reads=path_to_ccs_reads
out_dir=path_to_out_directory

/usr/local/packages/flye-2.9/bin/flye --pacbio-hifi $ccs_reads --genome-size 33m --out-dir $out_dir -t 16
```

   ### Using Canu

```
ccs_reads=path_to_ccs_reads

/usr/local/packages/canu-2.1.1/bin/canu -assemble -pacbio-hifi $ccs_reads
```
### Geneal coverage
```
samtools depth -a Reads2assembly.bam | awk '{sum+=$3} END {print sum/NR}'
```
## Assembly quality assessment 
Making genome assembly of Ld1S2D and further comparison with BPK282A2 (TritrypDB v63)
   ## Quast
```
ref_gen=path_to_ref_genome
ref_gff=path_to_ref_gff
out_dir=path_to_new_directory
assembly=path_to_assembly

/usr/local/packages/quast-5.2.0/quast.py -r $ref_gen -g $ref_gff -o $out_dir --min-contig 1000 -t 4 --gene-finding $assembly
```

   ## Mummer
```
ref_gen=path_to_ref_genome
assembly=path_to_assembly

/usr/local/packages/mummer-3.23/nucmer -p Ld1Svs282_WG_nucmer --mum $ref_gen $assembly

/usr/local/packages/mummer-3.23/mummerplot --png --filter --color --layout --prefix=Ld1Svs282_WG_nucmer Ld1Svs282_WG_nucmer.delta -R $ref_gen -Q $assembly
```
   ## Rearrangement
```
/usr/local/packages/mummer-3.23/show-diff Ld1Svs282_WG_nucmer.delta > Ld1Svs282_WG_nucmer.rearrangement
```
   ## Repeats
```
ref_gen=path_to_ref_genome
assembly=path_to_assembly

/usr/local/packages/mummer-3.23/nucmer --maxmatch --nosimplify --prefix=Ld1Svs282_WG_nucmermax $ref_gen $assembly

/usr/local/packages/mummer-3.23/show-coords -r Ld1Svs282_WG_nucmermax.delta > Ld1Svs282_WG_nucmermax.coords
```
## SNP density

MUMmer SNP comparition
```
ref_gen=path_to_ref_genome
assembly=path_to_assembly

/usr/local/packages/mummer-3.23/nucmer -mum $ref_gen $assembly

/usr/local/packages/mummer-3.23/delta-filter -r -q Ld1Svs282_WG_nucmer.delta > Ld1Svs282_WG_nucmer.filter

/usr/local/packages/mummer-3.23/show-snps -Clr Ld1Svs282_WG_nucmer.filter > Ld1Svs282_WG_nucmer.snps
```
Filtering Ld1Svs282_WG_nucmer.snps to extract only SNPs (no insertion, no deletion)
```
tail -n +6 Ld1Svs282_WG_nucmer.snps | awk '{if($2!=".") print}' | awk '{if($3!=".") print $1"\t"$14}' > SNP_density.tsv
```
Importing the dataset into R to plot the density for all chromosomes
```
#!/usr/bin/env Rscript

library(ggplot2)

colnames(SNP_density) <- c("Position", "Chr")
SNP_density$Position <- as.numeric(SNP_density$Position)

head(SNP_density)

ggplot(aes(Position, group = Chr), data=SNP_density) +
    geom_histogram(binwidth = 5000) + 
    #facet_wrap(~Chr, scales = "free") +
    scale_x_continuous(labels = function(x) format(x, scientific = TRUE))
```
## Local gene copy number variation using base converage

   ### Base coverage using mpileup
```   
ref_gen=path_to_ref_genome
BAM__RefAligned=path_to_BAM_file (from CCS reads aligned to the ref genome)

samtools mpileup $ref_gen $BAM__RefAligned
```
   ### Extracting data from the mpileup file
```
awk '{print $1"\t"$2"\t"$4}' Ld1Svs282_mpileup.txt > Ld1Svs282_PosCov.tsv

grep Ld01_v1s1 Ld1Svs282_PosCov.tsv > Ld1_cov.tsv
```
   ### Plotting coverage chromosome per chromosome with R
```
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
```

### tRNA detection
```
assembly=path_to_assembly

/usr/local/packages/trnascan-se-2.0.3/bin/eufindtRNA -r $assembly > Ld1S_tRNA_strict.csv 
```

## BUSCO
```
export PATH="/usr/local/packages/augustus-3.4.0/bin:$PATH"
export PATH="/usr/local/packages/augustus-3.4.0/scripts:$PATH"
export AUGUSTUS_CONFIG_PATH="/usr/local/packages/augustus-3.4.0/configs/"
export PATH="/usr/local/packages/metaeuk-6-a5d39d9/bin:$PATH"


assembly=path_to_assembly
out_dir=path_to_output_directory

busco -m genome -i $assembly --auto-lineage-euk --long -o $out_dir  -f
```
