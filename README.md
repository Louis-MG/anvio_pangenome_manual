# anvio_pangenome_manual
Describes all the steps needed to go from a fasta to an annotated pangenome with anvio.

# step 1 : prepare the fasta.

The fasta files must have a proper formating before being treated. This is simply about removing special characters from the headers. Once formated, the files can be used as `-fasta`.

* tool : [anvi-script-reformat-fasta](https://anvio.org/help/7.1/programs/anvi-script-reformat-fasta/)
* object documentation : https://anvio.org/help/7.1/artifacts/contigs-fasta/

```bash
for species in $(readlink -f /mnt/projects_tn03/metapangenome/DATA/species/*); do folder=$(basename "$species"); for genome in $(readlink -f "$species"/*.fna); do contigs=$(basename "$genome"); contigs="${contigs//\.fna/}"; anvi-script-reformat-fasta "$genome" -o "$folder"/"$contigs"_reformated.fasta --simplify-names ; done; done
```


# step 2 : transform the fasta file into a contig database

The genome-storage database is the transformed fasta file into a SQL object. It is recquired by every function of anvio and is its foundation.

* tool : [anvi-gen-contigs-database](https://anvio.org/help/main/programs/anvi-gen-contigs-database/)
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

Malssezia are fungi and thus would be poorly annotated using prodigal. Download the '.gbff' file of their genomes on RefSeq and get the right files from it with :
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

## align

```bash
mmseqs creatdb ./*genes-aa.faa ANNOTDB --db-type 1
mmseqs createdb uniref90
mmseqs search ANNOTDB uniref90DB RESULTSDB ~/tmp/ --search-type 3 --threads 120 blablabla 
```

## extract alignements of each genomes 

```bash
mmseqs convertalis ANNOTDB uniref90DB RESULTSDB RESULTSDB.m8
```

```bash
#for all the species folder, for all the gene-seq.fa files :
for i in $(readlink -f ./*/*.fa) ; do echo $i; j=${i//*GCF/GCF}; echo $j; k=${i//gene-seq.fa/align.m8}; echo $k; if [ ! -f "$k" ] ; then grep -F "$j" ./RESULTS_4_ANNOT/annot_results.m8 > "$k"; fi; done
```

## retain only one function per gene per genome :

```bash
bash annot.sh -i /path/to/species/folder/ -f /path/to/uniref_to_interpro_func.tsv 
#parallel alternative :
parallel bash annot.sh -i {} -f /path/to/uniref_to_interpro_func.tsv ::: $(readlink -f /path/to/species/folder*)
```

## add the anontation to the contigsDB :

```bash
for contigs in $(readlink -f /mnt/ssd/LM/results/contigsDB/*/*); do echo $contigs; i=${contigs//\/mnt\/ssd\/LM\/results\/contigsDB\//\/mnt\/projects_tn01\/metapangenome\/DATA\/results\/gene-fasta\/}; echo "$i"; functions=${i//-CONTIGS.db/_functions.tsv}; echo "$functions"; singularity run --bind '/mnt/ssd/LM/,/mnt/projects_tn01/metapangenome/' /mnt/projects_tn01/metapangenome/tools/anvio7.sif anvi-import-functions -i "$functions" -c "$contigs" --drop-previous-annotations ; done
```

# step 4 : produce the genomes storages 

Once all contigs.db are annotated, we can produce an external-genomes.txt file for each species, which will contain all the paths to contigs.

* tool : [anvi-script-gen-gen-genomes-file](https://anvio.org/help/main/programs/anvi-script-gen-genomes-file/)
* object documentation : https://anvio.org/help/main/artifacts/external-genomes/

```bash
mkdir /path/to/genomesDB
mkdir /path/to/genomesDB/species*
for i in * ; do if [ -d $i ]; then echo $i; echo /mnt/ssd/LM/results/genomesDB/"$i"/external_genomes.txt; echo "name" > ~/names.txt ; echo "contigs_db_path" > ~/paths.txt; for j in $(readlink -fe "$i"/*CONTIGS.db); do echo "$j" >> ~/paths.txt; name=$(basename "$j"); echo "$name" >> ~/names.txt; sed -i 's/-CONTIGS\.db//g;s/[-\.]/_/g' ~/names.txt ; paste ~/names.txt ~/paths.txt > /mnt/ssd/LM/results/genomesDB/"$i"/external_genomes.txt ; done ; fi ; done
# next :
for i in * ; do if [ -d $i ]; then singularity run --bind '/mnt/ssd/LM/,/mnt/projects_tn01/metapangenome/' /mnt/projects_tn01/metapangenome/tools/anvio7.sif anvi-gen-genomes-storage -o /mnt/ssd/LM/results/genomesDB/"$i"-GENOMES.db -e /mnt/ssd/LM/results/genomesDB/"$i"/external_genomes.txt ; fi ; done
```

# step 5 : produce the species pangenomes

The pangenome is computed with the annotations :

* tool : [anvi-pan-genome](https://anvio.org/help/main/programs/anvi-pan-genome/)
* object documentation : https://anvio.org/help/main/artifacts/pan-db/

```bash
#make the pangenomes through a loop :
for i in $(readlink -f ./*/*-GENOMES.db) ; do echo $i; j=$(basename "$i"); singularity run --bind '/mnt/ssd/LM/,/mnt/projects_tn03/metapangenome/' /mnt/projects_tn03/metapangenome/tools/anvio7.sif anvi-pan-genome --genomes-storage "$i" --project-name "${j//-GENOMES\.db/_pangenome}" ; done
```

Visualise the pangenome :

* tool : [anvi-display-pan](https://anvio.org/help/main/programs/anvi-display-pan/)
* object documentation : https://anvio.org/help/main/artifacts/interactive/

# step 6 : produce the metapangenome :

First we make an external_genomes.tsv file :

```bash
cd /path/to/contigsDB/
echo "name" > ~/names.txt ; echo "contigs_db_path" > ~/paths.txt; for i in * ; do if [ -d $i ]; then echo $i; for j in $(readlink -fe "$i"/*CONTIGS.db); do echo "$j" >> ~/paths.txt; name=$(basename "$j"); echo "$i"_"$name" >> ~/names.txt; sed -i 's/-CONTIGS\.db//g;s/[-\.]/_/g' ~/names.txt ; done ; fi ; done ; paste ~/names.txt ~/paths.txt > /mnt/ssd/LM/results/genomesDB/ALL_external_genomes.txt
```

Then we can build the genomes storage : 

```bash
singularity run --bind '/mnt/ssd/LM/,/mnt/projects_tn01/metapangenome/' /mnt/projects_tn01/metapangenome/tools/anvio7.sif anvi-gen-genomes-storage -o /mnt/ssd/LM/results/genomesDB/ALL-GENOMES.db -e /mnt/ssd/LM/results/genomesDB/ALL_external_genomes.txt
```

ANd use it to build the metapangenome :

```bash
singularity run --bind '/mnt/ssd/LM/,/mnt/projects_tn03/metapangenome/' /mnt/projects_tn03/metapangenome/tools/anvio7.sif anvi-pan-genome --genomes-storage "$i" --project-name "${j//-GENOMES\.db/_pangenome}"
```

