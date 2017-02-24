### ClinVar

#### In 1 sentence

This repo provides tools to convert ClinVar data into a tab-delimited flat file, and also provides that resulting tab-delimited flat file.

#### Output Files

The full set of tables generated by the pipeline for the latest clinvar release is stored in the [output](output/) folder.
These tables are separated by genome build ([output/b37](output/b37) or [output/b38](output/b38)) and whether they represent 
simple mono-allelic variants (`../single/` sub-directory), 
or more complex variants such as compound-heterozygotes, haplotypes, etc. (`../multi/` sub-directory) - for a total of 4 directories that contain tables.  
 
Each of these directories contains the following files:
* __clinvar_allele_trait_pairs.*.tsv.gz__: table where each row represents an allele-trait pair.
* __clinvar_alleles.*.tsv.gz__: table where each row represents a single variant allele. This is generated by grouping _clinvar_allele_trait_pairs.tsv.gz_ by allele.
* __clinvar_alleles.*.vcf.gz__: _clinvar_alleles.tsv.gz_ converted to VCF format.
* __clinvar_alleles_with_exac_v1.*.tsv.gz__: _clinvar_alleles.tsv.*.gz_ with additonal columns from the [ExAC v1](http://exac.broadinstitute.org/about) dataset that have non-empty values for all clinvar alleles that are also present in ExAC.  
* __clinvar_alleles_stats.*.txt__:  summary of the different columns in _clinvar_alleles.*.tsv.gz_, along with some basic stats on the different values that appear in each column.


#### Motivation

[ClinVar](http://www.ncbi.nlm.nih.gov/clinvar/) is a public database hosted by NCBI for the purpose of collecting assertions as to genotype-phenotype pairings in the human genome. One common use case for ClinVar is as a catalogue of genetic variants that have been reported to cause Mendelian disease. In our work in the [MacArthur Lab](http://macarthurlab.org/), we have two major use cases for ClinVar:

1. To check whether candidate causal variants we find in Mendelian disease exomes have been previously reported as pathogenic.
2. To pair with [ExAC](http://exac.broadinstitute.org/) data to enable exome-wide analyses of reportedly pathogenic variants.

ClinVar makes its data available via [FTP](ftp://ftp.ncbi.nlm.nih.gov/pub/clinvar/) in three formats: XML, TXT, and VCF. We found that none of these files were ideally suited for our purposes. The VCF only contains variants present in dbSNP; it is not a comprehensive catalogue of ClinVar variants. The TXT file lacks certain annotations such as PubMed IDs for related publications. The XML file is large and complex, with multiple entries for the same genomic variant, making it difficult to quickly look up a variant of interest. In addition, both the XML and TXT representations are not guaranteed to be unique on genomic coordinates, and also contain many genomic coordinates that have been parsed from HGVS notation, and therefore may be right-aligned (in contrast to left alignment, the standard for VCF) and may also be non-minimal (containing additional nucleotides of context to the left or right of a given variant).

#### Solution

To create a flat representation of ClinVar suited for our purposes, we took several steps, encapsulated in the pipeline [src/master.py](src/master.py) 
(which supercedes the older [src/master.bash](src/master.bash) script):

1. Download the latest XML and TXT dumps from ClinVar FTP.
2. Parse the XML file using [src/parse_clinvar_xml.py](src/parse_clinvar_xml.py) to extract fields of interest into a flat file.
4. Normalize using [our Python implementation](https://github.com/ericminikel/minimal_representation/blob/master/normalize.py) of [vt normalize](http://genome.sph.umich.edu/wiki/Variant_Normalization) (see [[Tan 2015]]).  
5. Group the allele-trait records by allele using [src/group_by_allele.py](src/group_by_allele.py) to aggregate interpretations from multiple submitters by allele, independent of conditions. 
5. Join the TXT file using [src/join_data.R](src/join_data.R) to aggregate interpretations from multiple submitters independent of conditions. 
6. Generate the VCF file and other tables based on the file created in 5.
  

&dagger;Because a ClinVar record may contain multiple assertions of Clinical Significance, we defined three additional columns:

+ `pathogenic` is `1` if the variant has *ever* been asserted "Pathogenic" or "Likely pathogenic" by any submitter for any phenotype, and `0` otherwise
+ `benign` is `1` if the variant has *ever* been asserted "Benign" or "Likely benign" by any submitter for any phenotype, and `0` otherwise
+ `conflicted` is `1` if the variant has *ever* been asserted "Pathogenic" or "Likely pathogenic" by any submitter for any phenotype, and has also been asserted "Benign" or "Likely benign" by any submitter for any phenotype, and `0` otherwise. Note that having one assertion of pathogenic and one of uncertain significance does *not* count as conflicted for this column. 

#### Dependancies

Enviroment must contain these executables wget, Rscript, tabix, vt: 
R, [htslib](https://github.com/samtools/htslib) 

tabix [How To Install](http://genometoolbox.blogspot.com/2013/11/installing-tabix-on-unix.html) is already apart of the samtools software package.

vt [vt](https://github.com/atks/vt) (make sure the `vt` command is callable, i.e. in your `$PATH`) [wkik](http://genome.sph.umich.edu/wiki/Vt).


#### Usage

To run the pipeline:

```
cd ./src
pip install --user --upgrade -r requirements.txt
python2.7 master.py --b37-genome /path/to/b37.fasta --b38-genome /path/to/b38.fasta -E /path/to/ExAC.r1.sites.vep.vcf.gz  
```

See `python master.py -h` for additional options.  


#### Usage notes

Because ClinVar contains a great deal of data complexity, we made a deliberate decision to *not* attempt to capture all fields in our resulting file. We made an effort to capture a subset of fields that we believed would be most useful for genome-wide filtering, and also included `measureset_id` as a column to enable the user to look up additional details on the ClinVar website. For instance, the page for the variant with `measureset_id` 7105 is located at [ncbi.nlm.nih.gov/clinvar/variation/7105/](http://www.ncbi.nlm.nih.gov/clinvar/variation/7105/). Note that we also do not capture all of the complexity of the fields that are included. 

If you want to analyze the output file in R, a suitable line of code to read it in would be:

```r
clinvar = read.table('clinvar_alleles.tsv',sep='\t',header=T,quote='',comment.char='')
```

#### License, terms, and conditions

ClinVar data, as a work of the United States federal government, are in the public domain and are redistributed here under [the same terms](http://www.ncbi.nlm.nih.gov/clinvar/docs/maintenance_use/) as they are distributed by ClinVar itself. Importantly, note that ClinVar data are "not intended for direct diagnostic use or medical decision-making without review by a genetics professional". The code in this repository is distributed under an MIT license.

[Tan 2015]: http://www.ncbi.nlm.nih.gov/pubmed/25701572 "Tan A, Abecasis GR, Kang HM. Unified representation of genetic variants. Bioinformatics. 2015 Jul 1;31(13):2202-4. doi: 10.1093/bioinformatics/btv112. Epub 2015 Feb 19. PubMed PMID: 25701572."
