# anvio_pangenome_manual
Describes all the steps needed to go from a fasta to an annotated pangenome with anvio.

# step 1 : prepare the fasta.

The fasta files must have a proper formating before being treated. This is simply about removing special characters from the headers. Once formated, the files can be used as `-fasta`.

* tool : [anvi-script-reformat-fasta](https://anvio.org/help/7.1/programs/anvi-script-reformat-fasta/)
* object documentation : https://anvio.org/help/7.1/artifacts/contigs-fasta/

```bash
declare -i counter=0
total_counter=$(readlink -f ncbi_dataset/data/*/*.fna | wc -l)
for i in $(readlink -f ncbi_dataset/data/*/*fna) ; do ++counter; filename="${i###/}"; echo -e "$counter/$total_counter" ; singularity run anvio7.sif anvi-script-reformat-fasta -f $i -o ncbi_datasets/$filename --simplify-names --seq-type NT; cat ncbi_datasets/$filename >> ncbi_datasets/BigFasta.fasta ; done
```


# step 2 : transform the fasta file into a contig database

The genome-storage database is the transformed fasta file into a SQL object. It is recquired by every function of anvio and is its foundation.

* tool : [anvi-gen-contigs-database](https://anvio.org/help/main/programs/anvi-gen-contigs-database/)
* object documentation : https://anvio.org/help/main/artifacts/contigs-db/

```bash
```

# step 3 : produce the annotation

Several types of annotations can be performed with various databases: kegg, pfam, cog etc.
I annotated by aligning the gene-seq.fa against Uniref90.

## extract the gene sequences

```bash
gene-seq
```

## align

```bash
mmseqs creatdb ./*gene-seq.fasta ANNOTDB --db-type 3
mmseqs createdb uniref90
mmseqs search ANNOTDB uniref90DB RESULTSDB ~/tmp/ --search-type 3 --threads 120 blablabla 
```

## extract alignements of each genomes 

```bash
mmseqs convertalis 
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

```

# step 5 : produce the pangenome

The pangenome is computed with the annotations :

* tool : [anvi-pan-genome](https://anvio.org/help/main/programs/anvi-pan-genome/)
* object documentation : https://anvio.org/help/main/artifacts/pan-db/

Visualise the pangenome :

* tool : [anvi-display-pan](https://anvio.org/help/main/programs/anvi-display-pan/)
* object documentation : https://anvio.org/help/main/artifacts/interactive/

