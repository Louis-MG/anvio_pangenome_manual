# anvio_pangenome_manual
Describes all the steps needed to go from a fasta to an annotated pangenome with anvio.

# step 1 : prepare the fasta.

The fasta files must have a proper formating before being treated. This is simply about removing special characters from the headers. Once formated, the files can be used as `-fasta`.

* tool : [anvi-script-reformat-fasta](https://anvio.org/help/8/programs/anvi-script-reformat-fasta/)
* object documentation : https://anvio.org/help/7.1/artifacts/contigs-fasta/

```bash
for species in $(readlink -f /mnt/projects_tn03/metapangenome/DATA/species/*); do folder=$(basename "$species"); for genome in $(readlink -f "$species"/*.fna); do contigs=$(basename "$genome"); contigs="${contigs//\.fna/}"; anvi-script-reformat-fasta "$genome" -o "$folder"/"$contigs"_reformated.fasta --simplify-names ; done; done
```


# step 2 : transform the fasta file into a contig database

The genome-storage database is the transformed fasta file into a SQL object. It is recquired by every function of anvio and is its foundation.

* tool : [anvi-gen-contigs-database](https://anvio.org/help/8/programs/anvi-gen-contigs-database/)
* object documentation : https://anvio.org/help/main/artifacts/contigs-db/

```bash
for species in $(readlink -f /mnt/projects_tn03/metapangenome/DATA/results/reformated/*); do folder=$(basename "$species"); for formated_genome in $(readlink -f "$species"/*reformated.fasta); do contigs=$(basename "$formated_genome"); contigs="${contigs//reformated.fasta/contigs-db.db}"; anvi-gen-contigs-database -f "$formated_genome" -o /mnt/ssd/LM/results/contigsDB/"$folder"/"$contigs" --project-name "$folder"_contigs -T 80 --force-overwrite --tmp-dir /mnt/ssd/LM/temp/ ; done ; done
```

# step 3 : produce the annotation

Several types of annotations can be performed with various databases: kegg, pfam, cog etc.
I annotated by aligning the gene-seq.fa against Uniref90.

## extract the gene sequences

```bash
for species in $(readlink -f /mnt/ssd/LM/results/contigsDB/*); do folder=$(basename "$species"); for contig_db in $(readlink -f "$species"/*.db); do contigs=$(basename "$contig_db"); output_name="${contigs//contigs-db.db/genes-aa.faa}"; anvi-get-sequences-for-gene-calls -c "$contig_db" --get-aa-sequences -o /mnt/ssd/LM/results/proteins/"$folder"/"$output_name" ; done ; done
```

## Special treatment of fungi

Malassezia are fungi and thus would be poorly annotated using prodigal. Download the '.gbff' file of their genomes on RefSeq and get the right files from it with :
```bash
anvi-script-process-genbank -i malassezia_restricta_genomic.gbff \
                            --output-gene-calls malassezia_restricta_gene_calls.tsv \
                            --output-functions malassezia_restricta_functions.tsv \
                            --output-fasta malassezia_restricta_refs.fa \
                            --annotation-source augustus


```
For _M. restricta_, report should be :

```
Num GenBank entries processed ................: 10
Num gene records found .......................: 4,406
Num genes reported ...........................: 3,210
Num genes with AA sequences ..................: 3,210
Num genes with functions .....................: 3,210
Locus tags included in functions output? .....: No
Num partial genes ............................: 0
Num genes excluded ...........................: 1,196
```
The 1,196 genes are exlueded for 'spanning multiple contigs'. For _M.globosa_ :

```
Num GenBank entries processed ................: 62
Num gene records found .......................: 4,278
Num genes reported ...........................: 3,107
Num genes with AA sequences ..................: 3,098
Num genes with functions .....................: 3,107
Locus tags included in functions output? .....: No
Num partial genes ............................: 9
Num genes excluded ...........................: 1,171
```
Again, 1171 genes were exlueded for the reason above. 9 partial genes were also exclueded.

Then Build the contigs databases using the files produced by the command above :

```bash
anvi-gen-contigs-database -f ../annot_malassezia/malassezia_globosa_refs.fa \
                          -o /mnt/ssd/LM/results/contigsDB/MalasseziaGlobosa/GCF_000181695.2_ASM18169v2_genomic_contigs-db.db \
                          -n MalasseziaGlobosa \
                          --external-gene-calls ../annot_malassezia/malassezia_globosa_gene_calls.tsv \
                          --split-length -1

anvi-gen-contigs-database -f ../annot_malassezia/malassezia_restricta_refs.fa
                           -o /mnt/ssd/LM/results/contigsDB/MalasseziaRestricta/GCF_003290485.1_ASM329048v1_genomic_contigs-db.db
                           -n MalasseziaRestricta
                           --external-gene-calls ../annot_malassezia/malassezia_restricta_gene_calls.tsv
                           --split-length -1
```

And import the functions to complete the annotation:

```bash
anvi-import-functions -c /mnt/ssd/LM/results/contigsDB/MalasseziaRestricta/GCF_003290485.1_ASM329048v1_genomic_contigs-db.db -i ../annot_malassezia/malassezia_restricta_functions.tsv
anvi-import-functions -c /mnt/ssd/LM/results/contigsDB/MalasseziaGlobosa/GCF_003290485.1_ASM329048v1_genomic_contigs-db.db -i ../annot_malassezia/malassezia_globosa_functions.tsv
```


## Align the protein sequences on UNIREF90 to obtain hteir functions.

```bash
mmseqs createdb /mnt/ssd/LM/results/proteins/*/*.faa /mnt/scratch/LM/ANNOT_QUERY_DB/query_DB --dbtype 1
mmseqs createdb uniref90
mmseqs search /mnt/scratch/LM/ANNOT_QUERY_DB/query_DB /mnt/scratch/LM/KMER/UNIREF90/uniref90 /mnt/scratch/LM/RESULTS_ANNOT_DB/results_DB /mnt/ssd/LM/tmp/ --search-type 1 --start-sens 2 --sens-steps 3 -s 7 --threads 120 -a
```

## extract alignements of each genomes 

```bash
mmseqs convertalis /mnt/scratch/LM/ANNOT_QUERY_DB/query_DB /mnt/scratch/LM/KMER/UNIREF90/uniref90 /mnt/scratch/LM/RESULTS_ANNOT_DB/results_DB /mnt/scratch/LM/RESULTS_ANNOT_DB/RESULTS_DB.m8 --format
```

```bash
#script spearates alignments per qset ID (which are the different files used to build the query_DB)
#script then moves the files to species folders in output path
bash split_annot_species.sh -m8 /mnt/scratch/LM/RESULTS_ANNOT_DB/results.m8 -s /mnt/projects_tn03/metapangenome/DATA/species -o /mnt/projects_tn03/metapangenome/DATA/results/annot/
```
Files that were not aligned because dark magic can be found again with :
```bash
for species in $(readlink -f results/proteins/*) ; do folder=$(basename "$species"); for i in $(readlink -f "$species"/*.faa); do file=$(basename "$i"); annot=${file//_genomic_genes-aa.faa/.m8}; if [ ! -e results/annot/"$folder"/"$annot" ]; then echo "$i" >> to_be_query_again.txt; fi ; done ; done
```
In `to_be_query_again.txt` are the absolute paths to the protein files that should be annotated. Remove the Malassezia lines and then build again the DB with :
```bash
mmseqs createdb $(sed "s/\\n/ /g" /mnt/projects_tn03/metapangenome/DATA/to_be_query_again.txt) QUERY_AGAIN/query

```	

## retain only one function per gene per genome :

```bash
bash annot.sh -i /path/to/species/folder/ -f /path/to/uniref_to_interpro_func.tsv 
#parallel alternative :
parallel bash annot.sh -i {} -f /path/to/uniref_to_interpro_func.tsv ::: $(readlink -f /path/to/species/folder*)
```

## add the anontation to the contigsDB :

```bash
for contigs in $(readlink -f /mnt/ssd/LM/results/contigsDB/*/*); do echo $contigs; i=${contigs//\/mnt\/ssd\/LM\/results\/contigsDB\//\/mnt\/projects_tn03\/metapangenome\/DATA\/results\/gene-fasta\/}; echo "$i"; functions=${i//-CONTIGS.db/_functions.tsv}; echo "$functions"; anvi-import-functions -i "$functions" -c "$contigs" --drop-previous-annotations ; done
```

# step 4 : produce the genomes storages 

Once all contigs.db are annotated, we can produce an external-genomes.txt file for each species, which will contain all the paths to contigs. Then, build the genome storages.
NOTE: eukaryotes and prokaryotes must be separated, otherwise anvioi will remove 'augustus gene calls'

* tool : [anvi-script-gen-gen-genomes-file](https://anvio.org/help/8/programs/anvi-script-gen-genomes-file/)
* object documentation : https://anvio.org/help/main/artifacts/external-genomes/

```bash
mkdir /path/to/genomesDB
mkdir /path/to/genomesDB/species*
for i in * ; do if [ -d $i ]; then echo $i; echo /mnt/ssd/LM/results/genomesDB/"$i"/external_genomes.txt; echo "name" > ~/names.txt ; echo "contigs_db_path" > ~/paths.txt; for j in $(readlink -fe "$i"/*CONTIGS.db); do echo "$j" >> ~/paths.txt; name=$(basename "$j"); echo "$name" >> ~/names.txt; sed -i 's/-CONTIGS\.db//g;s/[-\.]/_/g' ~/names.txt ; paste ~/names.txt ~/paths.txt > /mnt/ssd/LM/results/genomesDB/"$i"/external_genomes.txt ; done ; fi ; done
# next :
for i in * ; do if [ -d $i ]; then anvi-gen-genomes-storage -o /mnt/ssd/LM/results/genomesDB/"$i"-GENOMES.db -e /mnt/ssd/LM/results/genomesDB/"$i"/external_genomes.txt ; fi ; done
```

# step 5 : produce the species pangenomes

The pangenome is computed with the annotations :

* tool : [anvi-pan-genome](https://anvio.org/help/8/programs/anvi-pan-genome/)
* object documentation : https://anvio.org/help/main/artifacts/pan-db/

```bash
#make the pangenomes through a loop :
for i in $(readlink -f ./*/*-GENOMES.db) ; do echo $i; j=$(basename "$i"); anvi-pan-genome --genomes-storage "$i" --project-name "${j//-GENOMES\.db/_pangenome}" ; done
```

Visualise the pangenome :

* tool : [anvi-display-pan](https://anvio.org/help/8/programs/anvi-display-pan/)
* object documentation : https://anvio.org/help/main/artifacts/interactive/

# step 6 : produce the metapangenome :

First we make an external_genomes.tsv file :

```bash
cd /path/to/contigsDB/
echo "name" > ~/names.txt ; echo "contigs_db_path" > ~/paths.txt; for i in * ; do if [ -d $i ]; then echo $i; for j in $(readlink -fe "$i"/*CONTIGS.db); do echo "$j" >> ~/paths.txt; name=$(basename "$j"); echo "$i"_"$name" >> ~/names.txt; sed -i 's/-CONTIGS\.db//g;s/[-\.]/_/g' ~/names.txt ; done ; fi ; done ; paste ~/names.txt ~/paths.txt > /mnt/ssd/LM/results/genomesDB/ALL_external_genomes.txt
```

Then we can build the genomes storage : 

```bash
anvi-gen-genomes-storage -o /mnt/ssd/LM/results/genomesDB/ALL-GENOMES.db -e /mnt/ssd/LM/results/genomesDB/ALL_external_genomes.txt
```

And use it to build the metapangenome :

```bash
anvi-pan-genome --genomes-storage "$i" --project-name "${j//-GENOMES\.db/_pangenome}"
```

##### For the metagenomics reads recruitment:

# Fungi:

# step 1 : align the reads to the genomes

```bash
#build the bowtiedb, add the annotation to contigs
cat Malassezia_*_genomes.fna >> fungi.fna
#import the bgff: it will output a fungi_refs.fa to use instead of the fungi.fna, a fungi_gunctions.txt to import. Otherwise anvio doesnt work.
bowtie2-build fungi_refs.fa
anvi-ben-contigs-database -f fungi_refs.fa -o fungi_contigs-db.db --external-gene-calls fungi_refs_gene_calls.tsv
anvi-import-functions -c fungi_contigs-db.db -i fungi_functions.txt

# align the reads for different skin env
readlink -f /mnt/projects_tno03/metapangenome/DATA/Biogeography/dry/*_clean_R1* > bowtie/dry/samples.txt
sed -i 's/_R1.fastq.gz//g' bowtie/dry/samples.txt
while read path ; do  sample=$(basename "$path") ;  if [ "$sample" == "sample" ]; then continue; fi;     bowtie2 --threads 60             -x fungi_bowtie_DB             -1 "$path"_R1.fastq.gz             -2 "$path"_R2.fastq.gz             --no-unal             -S bowtie/dry/"$sample".sam;      samtools view -F 4 -bS bowtie/dry/"$sample".sam > bowtie/dry/"$sample"-RAW.bam;      samtools sort bowtie/dry/"$sample"-RAW.bam -o bowtie/dry/"$sample".bam;     samtools index bowtie/dry/"$sample".bam;      rm bowtie/dry/"$sample".sam bowtie/dry/"$sample"-RAW.bam; done < bowtie/dry/samples.txt

readlink -f /mnt/projects_tno03/metapangenome/DATA/Biogeography/moist/*_clean_R1* > bowtie/dry/samples.txt
sed -i 's/_R1.fastq.gz//g' bowtie/moist/samples.txt
while read path ; do  sample=$(basename "$path") ;  if [ "$sample" == "sample" ]; then continue; fi;     bowtie2 --threads 60             -x fungi_bowtie_DB             -1 "$path"_R1.fastq.gz             -2 "$path"_R2.fastq.gz             --no-unal             -S bowtie/moist/"$sample".sam;      samtools view -F 4 -bS bowtie/moist/"$sample".sam > bowtie/moist/"$sample"-RAW.bam;      samtools sort bowtie/moist/"$sample"-RAW.bam -o bowtie/moist/"$sample".bam;     samtools index bowtie/moist/"$sample".bam;      rm bowtie/moist/"$sample".sam bowtie/moist/"$sample"-RAW.bam; done < bowtie/moist/samples.txt

readlink -f /mnt/projects_tno03/metapangenome/DATA/Biogeography/dry/*_clean_R1* > bowtie/dry/samples.txt
sed -i 's/_R1.fastq.gz//g' bowtie/dry/samples.txt
while read path ; do  sample=$(basename "$path") ;  if [ "$sample" == "sample" ]; then continue; fi;     bowtie2 --threads 60             -x fungi_bowtie_DB             -1 "$path"_R1.fastq.gz             -2 "$path"_R2.fastq.gz             --no-unal             -S bowtie/dry/"$sample".sam;      samtools view -F 4 -bS bowtie/dry/"$sample".sam > bowtie/dry/"$sample"-RAW.bam;      samtools sort bowtie/dry/"$sample"-RAW.bam -o bowtie/dry/"$sample".bam;     samtools index bowtie/dry/"$sample".bam;      rm bowtie/dry/"$sample".sam bowtie/dry/"$sample"-RAW.bam; done < bowtie/dry/samples.txt
```

# make the profile db foreach env :

```bash
while read path ; do  sample=$(basename "$path") ;  if [ "$sample" == "sample" ]; then continue; fi;     anvi-profile -c fungi_contigs-db.db                  -i bowtie/sebaceous/$sample.bam                  -M 1000                  --skip-SNV-profiling                  --num-threads 30                  -o bowtie/sebaceous/$sample ; done < bowtie/sebaceous/samples.txt

while read path ; do  sample=$(basename "$path") ;  if [ "$sample" == "sample" ]; then continue; fi;     anvi-profile -c fungi_contigs-db.db                  -i bowtie/dry/$sample.bam                  -M 1000                  --skip-SNV-profiling                  --num-threads 30                  -o bowtie/dry/$sample ; done < bowtie/dry/samples.txt

while read path ; do  sample=$(basename "$path") ;  if [ "$sample" == "sample" ]; then continue; fi;     anvi-profile -c fungi_contigs-db.db                  -i bowtie/moist/$sample.bam                  -M 1000                  --skip-SNV-profiling                  --num-threads 30                  -o bowtie/moist/$sample ; done < bowtie/moist/samples.txt
```

# merge  the db for each skin type :

```bash
for env in dry moist sebaceous ; do anvi-merge bowtie/"$env"/*/PROFILE.db            -o bowtie/"$env"/"$env"-MERGED            -c fungi_contigs-db.db ; done
```

# make the genome collection :

```bash
for split_name in `sqlite3 fungi_contigs-db.db 'select split from splits_basic_info;'`; do GENOME=`echo $split_name | awk 'BEGIN{FS="_split_"}{print $1}'`; echo -e "$split_name\t$GENOME" ; done > fungi-GENOME-COLLECTION.txt
# find which contig_split is in which genome, thus here for the fungi which malassezia
```

# add the genome colleciton to eah profile.db, per skin type :

```bash
for env in dry moist sebaceous ; do anvi-import-collection fungi-GENOME-COLLECTION.txt -c fungi_contigs-db.db -C Genomes -p bowtie/"$env"/"$env"-MERGED/PROFILE.db ; done
```

# compute the coverage summary of the contigs for each skin type :

```bash
for env in dry moist sebaceous ; do anvi-summarize -c fungi_contigs-db.db -C Genomes -p bowtie/"$env"/"$env"-MERGED/PROFILE.db --init-gene-coverages -o fungi-"$env"-SUMMARY; done
```

# generate internal genomes tables :

```bash
# marche paaaaaaaaaaaaaaaaas faut que je change pour avoir un readlink -f vers le chemin de la PROFILE.db issue du merge des profile.db d'un env
bash ../../../../gitlab/anvi-script-gen-internal-genomes-table.sh -i fungi-dry-SUMMARY -c fungi_contigs-db.db -P bowtie/dry/dry-MERGED/PROFILE.db -o ./ -p dry
bash ../../../../gitlab/anvi-script-gen-internal-genomes-table.sh -i fungi-moist-SUMMARY -c fungi_contigs-db.db -P bowtie/moist/moist-MERGED/PROFILE.db -o ./ -p moist
bash ../../../../gitlab/anvi-script-gen-internal-genomes-table.sh -i fungi-sebaceous-SUMMARY -c fungi_contigs-db.db -P bowtie/sebaceous/sebaceous-MERGED/PROFILE.db -o ./ -p sebaceous
```

# generate GENOMES db for each env :

```bash
anvi-gen-genomes-storage -i dry-internal-genomes-table.txt -o fungi-dry-GENOMES.db --gene-caller augustus
anvi-gen-genomes-storage -i moist-internal-genomes-table.txt -o fungi-moist-GENOMES.db --gene-caller augustus
anvi-gen-genomes-storage -i sebaceous-internal-genomes-table.txt -o fungi-sebaceous-GENOMES.db --gene-caller augustus
```

# generate pangenome for each env

```bash
for site in dry sebaceous moist;
do
	anvi-pan-genome -o fungi-"$site"-PAN -g fungi-"$site"-GENOMES.db -n fungi-"$site" -T 30
done
```

# generate the meta-pan-genome for each env:

```bash
for site in dry sebaceous moist;
do
        anvi-meta-pan-genome -i "$site"-internal-genomes-table.txt  -g fungi-"$site"-GENOMES.db -p fungi-"$site"-PAN/*PAN.db --gene-caller augustus
done
```

####################
#
# enrich. fonction. 
#
####################

# dry vs toenail

```bash
for i in AcinetobacterBaumannii SalmonellaEnterica CutibacteriumAcnes CutibacteriumNamnetense ; do cat /mnt/ssd/LM/results/genomesDB/"$i"/external_genomes.txt >> /mnt/ssd/LM/results/functional_enrichment/external_genomes.txt ; done
for i in AcinetobacterBaumannii SalmonellaEnterica CutibacteriumAcnes CutibacteriumNamnetense ; do awk 'BEGIN {OFS="\t"} {print $1, "dry"}' /mnt/ssd/LM/results/genomesDB/"$i"/external_genomes.txt >> /mnt/ssd/LM/results/functional_enrichment/groups.txt ; done
for i in CorynebacteriumAurimucosum CorynebacteriumIhumii CorynebacteriumPseudogenitalium StaphylococcusEpidermidis BrevibacteriumPaucivorans CorynebacteriumJeikeium CorynebacteriumKefirresidentii Corynebacteriumsphmsc04h06 StaphylococcusPettenkoferi Staphylococcusspez-p03 StaphylococcusWarneri ; do cat /mnt/ssd/LM/results/genomesDB/"$i"/external_genomes.txt >> /mnt/ssd/LM/results/functional_enrichment//external_genomes.txt ; done
for i in CorynebacteriumAurimucosum CorynebacteriumIhumii CorynebacteriumPseudogenitalium StaphylococcusEpidermidis BrevibacteriumPaucivorans CorynebacteriumJeikeium CorynebacteriumKefirresidentii Corynebacteriumsphmsc04h06 StaphylococcusPettenkoferi Staphylococcusspez-p03 StaphylococcusWarneri ; do awk 'BEGIN {OFS="\t"} {print $1, "toenail"}' /mnt/ssd/LM/results/genomesDB/"$i"/external_genomes.txt >> /mnt/ssd/LM/results/functional_enrichment/groups.txt ; done
anvi-script-gen-function-matrix-across-genomes -e external_genomes.txt -G groups.txt --annotation-source Uniref90  --output-file-prefix dry_vs_toenail
```

# moist vs toenail

#TODO: a ecrire
