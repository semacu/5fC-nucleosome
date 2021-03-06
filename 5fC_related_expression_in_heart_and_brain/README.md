
# Compare 5fC / Nucleosome occupancy across heart/brain tissues, and its relation to gene expression

The following steps are required:

1.  Intersecting 5fC Heart with histone marks and CpGi
2.  Unique sites 5fC to Heart and Brain
3.  Nucleosomes occupancy in Heart vs Brain considering 5fC

and finally 

4.  Connect (differential) gene expression with 5fC/Nucleosome presence in enhancer regions for Heart vs Brain. Get me [there](gene_expression_5fC_nuc_heart_and_brain.md)!

### Intersecting 5fC Heart with histone marks and CpGi

`CRUK-clust1`

```bash
#! /usr/bin/env bash
cd /scratcha/sblab/portel01/project/euni_5fc/intersect_histone_marks_cpgi_heart

raw_data="/scratcha/sblab/portel01/project/euni_5fc/heart_chipseq_5fC_data"

wt_5fC=${raw_data}/5fC_heart_i9_WT.bed.gz


# add chr in front, required becase the annotations have it
chr_wt_5fC=chr_5fC_heart_i9_WT.bed.gz

zcat ${wt_5fC} | awk '{print "chr"$0}' | gzip > ${chr_wt_5fC}


# These are the ref names for the following histone modifications
# already after the lift-over
# the data was in uk-cri-lcst01:/lustre/sblab/martin03/repository/20151125_5fC_nucleosome/data/encode/recent_Ren2014/mm10
H3K27ac=H3K27ac.mm9.bed
H3K4me1=H3K4me1.mm9.bed
H3K4me3=H3K4me3.mm9.bed
H3K36me3=H3K36me3.mm9.bed
H3K27me3=H3K27me3.mm9.bed
# from here /scratcha/sblab/martin03/reference_data/genomes/mm9.cpgIslandExtUnmasked.bed
# prepared like https://github.com/sblab-bioinformatics/projects/blob/master/20151125_5fC_nucleosome/20170228_5fC_cpgi/20170228_5fC_cpgi.md#where-are-the-cpg-islands-for-mm9
CpGi=mm9.cpgIslandExtUnmasked.bed

for modif in H3K27ac H3K4me1 H3K4me3 H3K36me3 H3K27me3 CpGi
do

    # on for the overlap, and the other for the reverse/complementary
    # not very elegant, but works
    for method in overlap reverse
    do

    # using the variable as another variable is frowned upon, too bad
    working_bed=${!modif}

    if [ ${method} == reverse ]
    then

        # WT
        bedtools intersect  -v -b ${chr_wt_5fC}  -a ${working_bed} \
        | awk -v kk=${modif} '{printf"%s\t%d\t%d\t%s\n",$1,$2,$3,kk}' | gzip \
        > ${modif}_no5fC_heart_consensus_WT.bed.gz

    else

        # WT
        bedtools intersect -b ${chr_wt_5fC}  -a ${working_bed} -u -wa -f 0.4 -r   \
        | awk -v kk=${modif} '{printf"%s\t%d\t%d\t%s\n",$1,$2,$3,kk}' | gzip \
        > ${modif}_5fC_heart_consensus_WT.bed.gz

    fi

    done
done
```

### Unique sites to 5fC Heart and Brain

`CRUK-clust1`

```bash
#! /usr/bin/env bash

cd /scratcha/sblab/portel01/project/euni_5fc/difference_5fC_sites_heart_brain
path_brain_data="/scratcha/sblab/portel01/project/euni_5fc/chipseq_5fC_data/"

# unique to heart
bedtools intersect -a 5fC_heart_i9_WT.bed.gz \
-b $path_brain_data/grm069_TDG_WT_O3_hindbrain.clean_peaks.narrowPeak.gz \
$path_brain_data/grm070_TDG_WT_Q3_hindbrain.clean_peaks.narrowPeak.gz \
-v  | gzip > 5fC_unique_to_heart.bed.gz

# unique to brain
bedtools intersect -a 5fC_consensus_WT.bed.gz -b 5fC_heart_i9_WT.bed.gz -v  | gzip > 5fC_unique_to_brain.bed.gz


# common mnase sites
bedtools intersect -a heart_mnase_wt_j9.bed.gz -b heart_mnase_wt_kt.bed.gz -f 0.9 -r -wa -u | gzip  > heart_mnase_consensus_09r.bed.gz
```

### Nucleosomes occupancy in Heart vs Brain considering 5fC

The nucleosome peak calling was done by using iNPS, see [here](../MNase-seq/README.md).

This [notebook](scripts/nuc_occupancy_vs_5fC_heart_brain.ipynb) collects the data and calculates 
the statistical significance of difference 
between the two pairs of distributions. 


## Gene expression dependent on 5fC/Nucleosome presence Heart vs Brain

There are several steps required, first is to obtain the differential expression (described [here](../RNA-seq/README.md)).
Next, using the unique-to-heart, unique-to-brain, in combination with presence or
absence of nucleosomes, we want to relate these sites with enhancers, and
from there to gene expression differences.

#### New unique sites Brain vs Heart in WT

```bash

# Nucleosome sites unique to brain
bedtools intersect -a ConsensusNucleosome_iNPS_Brain_WT_noMT.bed -b ConsensusNucleosome_iNPS_Heart_WT_noMT.bed -v -f 0.8 -r >  Nuclesome_sites_unique_to_brain_wt.bed

# Nucleosome sites unique to heart

bedtools intersect -a ConsensusNucleosome_iNPS_Heart_WT_noMT.bed -b ConsensusNucleosome_iNPS_Brain_WT_noMT.bed -v -f 0.8 -r >  Nuclesome_sites_unique_to_hear_wt.bed
```

#### Nucleosome occupancy WT vs KO in Brain

Prepare differential nucleosome coverage in brain, using DANPOS. Please note that 
the paths are specific to the machine in which the analysis was done.

```bash
#! /usr/bin/env bash

wt_1="../../mnase_bam/ear030_HindbrainA1.150618.ncbi37.bam"
wt_2="../../mnase_bam/ear032_HindbrainF1.150618.ncbi37.bam"
ko_1="../../mnase_bam/ear040_K1_hindbrain.SLX-10219.ncbi37.bam"
ko_2="../../mnase_bam/ear041_K2_hindbrain.SLX-10219.ncbi37.bam"

if [ ! -d bams_wt ]
then
    mkdir bams_wt
fi

cp ${wt_1} bams_wt/wt_rep1.bam
cp ${wt_2} bams_wt/wt_rep2.bam

if [ ! -d bams_ko ]
then
    mkdir bams_ko
fi

cp ${ko_1} bams_ko/ko_rep1.bam
cp ${ko_2} bams_ko/ko_rep2.bam

sbatch -J DPOS --wrap "~/programs/danpos-2.2.2/danpos.py dpos bams_ko/:bams_wt/  -m 1 && rm -fr bams_wt bams_ko"
```

Extract the summit position and location of reference positions. We will
get one list for locations that are new to heart, and those that
are new for brain.

#### Nucleosome occupancy WT vs KO in Heart

Using `DANPOS`, same as for brain above (*mutatis mutandis*)

```bash
#! /usr/bin/env bash

wt_1="/scratcha/sblab/portel01/project/euni_5fc/heart_bam/ear038_K7_heart.SLX-10219.ncbi37.bam"
wt_2="/scratcha/sblab/portel01/project/euni_5fc/heart_bam/ear039_J9_heart.SLX-10219.ncbi37.bam"
ko_1="/scratcha/sblab/portel01/project/euni_5fc/heart_bam/ear036_K1_heart.SLX-10219.ncbi37.bam"
ko_2="/scratcha/sblab/portel01/project/euni_5fc/heart_bam/ear037_K2_heart.SLX-10219.ncbi37.bam"

if [ ! -d bams_wt ]
then
    mkdir bams_wt
fi

cp ${wt_1} bams_wt/wt_rep1.bam
cp ${wt_2} bams_wt/wt_rep2.bam

if [ ! -d bams_ko ]
then
    mkdir bams_ko
fi

cp ${ko_1} bams_ko/ko_rep1.bam
cp ${ko_2} bams_ko/ko_rep2.bam

sbatch -J DPOS --wrap "~/programs/danpos-2.2.2/danpos.py dpos bams_ko/:bams_wt/  -m 1 && rm -fr bams_wt bams_ko"
```

#### Nucleosome occupancy WT Brain vs KO Heart

Using `DANPOS`, same as above (*mutatis mutandis*)

```bash

brain_wt_1="/scratcha/sblab/portel01/project/euni_5fc/mnase_bam/ear030_HindbrainA1.150618.ncbi37.bam"
brain_wt_2="/scratcha/sblab/portel01/project/euni_5fc/mnase_bam/ear032_HindbrainF1.150618.ncbi37.bam"
heart_wt_1="/scratcha/sblab/portel01/project/euni_5fc/heart_bam/ear038_K7_heart.SLX-10219.ncbi37.bam"
heart_wt_2="/scratcha/sblab/portel01/project/euni_5fc/heart_bam/ear039_J9_heart.SLX-10219.ncbi37.bam"

if [ ! -d bams_brain ]
then
    mkdir bams_brain
fi

cp ${brain_wt_1} bams_brain/brain_wt_rep1.bam
cp ${brain_wt_2} bams_brain/brain_wt_rep2.bam

if [ ! -d bams_heart ]
then
    mkdir bams_heart
fi

cp ${heart_wt_1} bams_heart/heart_wt_rep1.bam
cp ${heart_wt_2} bams_heart/heart_wt_rep2.bam

sbatch -J DPOS --wrap "~/programs/danpos-2.2.2/danpos.py dpos bams_brain/:bams_heart/  -m 1 && rm -fr bams_brain bams_heart"
```

##  Differential expression related to enhancers with 5fC-nucleosomes Heart vs brain
The instructions/workflow are [here](gene_expression_5fC_nuc_heart_and_brain.md)
