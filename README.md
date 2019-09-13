### 

This repository contains the code used in the analyses for the paper "Expert Curation of the Human and Mouse Olfactory Receptor Gene Repertoires Identifies Conserved Coding Regions Split Across Two Exons", by If Barnes, Ximena Ibarra-Soria and collaborators.

### Transcriptome of human olfactory mucosa

The markdown document `human_ORexpression` contains the code used to analyse the bulk RNA-seq data from human biopsies of olfactory mucosa. 

- It produces normalised counts for all OR genes.
- Compares transcript length as a function of amount of sequencing data available.
- Analyses the split OR genes presented in the paper (Figure 5).

Data is available from the EGA under study [EGAS00001001486](https://www.ebi.ac.uk/ega/studies/EGAS00001001486).

Additional files required:

- `data/geneCounts_human.RAW.tsv` contains the matrix of raw counts from all 9 samples from study EGAS00001001486 (this paper and Saraiva et al., Sci Adv, 2019) plus three samples from Olender et al., BMC Genomics, 2016.
- OR expression estimates for the mouse data were taken from Ibarra-Soria et al., eLife, 2017, and can be downloaded from https://doi.org/10.7554/eLife.21476.006.
- `data/mouseOR_length.tsv` provides the length in bp for each mouse OR gene, calculated from the Ensembl-HAVANA curated gene models.
- `data/OR_intron_sizes_human.txt` and `data/OR_intron_sizes_mouse.txt` contain the length in bp of the introns for every protein-coding OR transcript in the human and mouse repertoires, respectively. Data is from the Ensembl-HAVANA curated gene models. The introns are always presented from 5' to 3', irrespective of gene strand. For genes without introns, the filed for 'intron1' is set to 0.

### Single-cell RNA-seq of mouse OSNs

The markdown document `singleCell_splitORexpression` contains the code used to analyse the single-cell RNA-seq data from 34 manually picked OSNs, obtained from heterozygous OMP-GFP mice, at postnatal day 3.

- It performs QC and normalisation.
- Analyses the expression of OR genes in each cell, and investigates mismapping events.
- Compares the two cells expressing split ORs to the rest that express intronless ORs.

Data is available from ArrayExpress under accession number [E-MTAB-8285](https://www.ebi.ac.uk/arrayexpress/experiments/E-MTAB-8285). The matrix of raw counts for all cells is provided as a processed file, and need to be downloaded to execute the `markdown` document.

Additional files required:

- `data/mappingStats_singleCells.tsv` contains the mapping statistics generated by `STAR`.
- `data/indices_singleCells.tsv` contains the indices used during library prep.
