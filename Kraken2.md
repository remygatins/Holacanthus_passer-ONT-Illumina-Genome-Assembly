Kraken2- remove contamination from assembly
================
Remy Gatins
June 01, 2020

-   [Install Kraken2](#install-kraken2)
-   [Build the kraken2 database](#build-the-kraken2-database)
-   [Run Kraken](#run-kraken)
    -   [Remove contamination from assembly](#remove-contamination-from-assembly)

KRAKEN2 is a taxonomic classification system using exact k-mer matches to achieve high accuracy and fast classification speeds. This classifier matches each k-mer within a query sequence to the lowest common ancestor (LCA) of all genomes containing the given k-mer. The k-mer assignments inform the classification algorithm. For more information go to <https://ccb.jhu.edu/software/kraken2/>

We are running Kraken to identify and discard contamination in our full assembly using the `--confidence` flag.

Install Kraken2
---------------

First navigate to where you want to install Kraken2.
\*the conda version of Kraken2 was not up to date (latest version v2.0.9) so I installed this manually

clone kraken2 from github page

``` bash
git clone https://github.com/DerrickWood/kraken2.git
```

and install kraken2 indicating the path you wish to install it in

``` bash
./install_kraken2.sh /hb/groups/bernardi_lab/programs/kraken2
```

Build the kraken2 database
--------------------------

you will need to run this using a bash script: kraken.mpi

``` bash
#!/bin/bash
#SBATCH --output=kraken.out
#SBATCH --error=kraken.err
#SBATCH --partition=128x24
#SBATCH --nodes=1
#SBATCH --job-name=kraken
#SBATCH --mail-type=ALL
#SBATCH --mail-user=rgatins@ucsc.edu
#SBATCH --mem=120GB
#SBATCH --ntasks-per-node=20

#--------------
#load programs
#--------------

#-------------
# run command
#-------------

/hb/groups/bernardi_lab/programs/kraken2/kraken2-build --standard --threads 20 --db /hb/groups/bernardi_lab/programs/kraken2/kraken2_db/kraken2_db --use-ftp --no-masking
```

The database took multiple days to fully build with these parameters. The connection got severed a few times having to re-start the build. In theory the build should start where ever it left off. In your directory you should now have `kraken2_db` which should include the following

``` bash
hash.k2d  library  opts.k2d  seqid2taxid.map  taxo.k2d  taxonomy
```

\*building the database is the most tedious part but once you have it, running kraken is fast.

**bernardi\_lab:** `kraken2_db` is found in `/hb/groups/bernardi_lab/programs/kraken2/kraken2_db`

Run Kraken
----------

    /hb/groups/bernardi_lab/programs/kraken2/kraken2 --db kraken2_db/ --report kraken_report --use-names --confidence --threads 20 kraken_remi/HPA_04_pilon.fasta

#### Remove contamination from assembly

**Observation**: here, we are keeping the out and error files separate. The output (log) file will have the actual output that we want.

Output files:

    kraken.err kraken_report kraken.out 

Since the full kraken2 output was saved in the log file, first we will get only the list of contigs:

    grep "ctg" kraken.log > kraken_results.txt

Now get a list of **human** and **unclassified** contigs:
By doing this, we are removing anything classified as bacteria, plasmids or viruses.
We keep the **unclassified** because they might contain real contigs.

    grep "unclassified|Homo sapiens" kraken_results.txt | cut -f2 > assembly.fa.HsU.list

Using samtools, we will extract from the assembly only the contigs

    grep "^C" kraken2_pdog_assembly_notHsapiens.txt | cut -f2 > kraken2_pdog_assembly_notHsapiens_contigs.txt

We will remove the contigs in this list from the original assembly. We had a total of 1308 contigs that were classified as not human - which means that they corresponded to bacteria, plasmids or viruses.
