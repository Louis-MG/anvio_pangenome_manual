# anvio_pangenome_manual
Describes all the steps needed to go from a fasta to an annotated pangenome with anvio.

# step 1 : prepare the fasta.

The fasta files must have a proper formating before being treated. This is simply about removing special characters from the headers. Once formated, the files can be used as `-fasta`.

* tool : [anvi-script-reformat-fasta](https://anvio.org/help/7.1/programs/anvi-script-reformat-fasta/)
* object documentation : https://anvio.org/help/7.1/artifacts/contigs-fasta/

# step 2 : transform the fasta file into a contig database

The genome-storage database is the transformed fasta file into a SQL object. It is recquired by every function of anvio and is its foundation.

* tool : [anvi-gen-genomes-storage](https://anvio.org/help/main/programs/anvi-gen-genomes-storage/)
* object documentation : https://anvio.org/help/main/artifacts/genomes-storage-db/

# step 3 : produce the annotation

Several types of annotations can be performed with various databases: kegg, pfam, cog etc.

### taxonomic annotation :

* tool : [anvi-run-scg-taxonomy](https://anvio.org/help/main/programs/anvi-run-scg-taxonomy/)
* object documentation : https://anvio.org/help/main/artifacts/scgs-taxonomy/

### functionnal annotation :

With a custom hmm :
* tool : [anvi-run-hmms](https://anvio.org/help/main/programs/anvi-run-hmms/)
* object documentation : https://anvio.org/help/main/artifacts/hmm-hits/

With pfam :
get the Pfam database : 
* tool : [anvi-setup-pfams](https://anvio.org/help/7.1/programs/anvi-setup-pfams/)
* object documentation : https://anvio.org/help/7.1/artifacts/pfams-data/
Use it to annotate :
* tool : [anvi-run-pfams](https://anvio.org/help/7.1/programs/anvi-run-pfams/)
* object documentation : https://anvio.org/help/7.1/artifacts/functions/

With kegg :
* tool : [anvi-run-kegg-kofams](https://anvio.org/help/7.1/programs/anvi-run-kegg-kofams/)
* object documentation : https://anvio.org/help/7.1/artifacts/kegg-functions/

With cogs:
get the cogs database :
* tool : [anvi-setup-ncbi-cogs](https://anvio.org/help/main/programs/anvi-setup-ncbi-cogs/)
* object documentation : https://anvio.org/help/main/artifacts/cogs-data/
Use it to annotate :
* tool : [anvi-run-ncbi-cogs](https://anvio.org/help/main/programs/anvi-run-ncbi-cogs/)
* object documentation : see functions

# step 4 : estimate completeness

Once the annotation is complete (taxonomic annotation of scgs mandatory), you can estimate the completness of the genomes to determine the quality of the pangenome.

* tool : [anvi-estimate-genome-completeness](https://anvio.org/help/main/programs/anvi-estimate-genome-completeness/)
* object documentation : https://anvio.org/help/main/artifacts/completion/

# step 5 : produce the pangenome

The pangenome is conmputed with the annotations :

* tool : [anvi-pan-genome](https://anvio.org/help/main/programs/anvi-pan-genome/)
* object documentation : https://anvio.org/help/main/artifacts/pan-db/

Visualise the pangenome :

* tool : [anvi-display-pan](https://anvio.org/help/main/programs/anvi-display-pan/)
* object documentation : https://anvio.org/help/main/artifacts/interactive/

