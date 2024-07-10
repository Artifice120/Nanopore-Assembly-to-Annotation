# Nanopore-Assembly-to-Annotation
Rough draft for a guide on one way to assembly and annotate nanopore reads 
------------
After a fastq file has been generated from raw reads with [Guppy](https://timkahlke.github.io/LongRead_tutorials/BS_G.html) there will probably be noise in the reads that can cause issues when the reads need to be mapped later on after assembly of the reads into contigs.
To avoid these issues it is good practice to remove these errors before assembling the reads, although it can also be done after assmebly if need be depending on the assembler that you use (Trinity and Canu don't have issues with noisy reads). I prefer to use the tool [Seq tk](https://github.com/lh3/seqtk) since it is a small module using their sanatize option as seen below: 
```
 seqkit sana /input/location.fastq -o /output/location.fastq
```
This command is not too demanding for CPU and memory so it can be ran on a laptop and will take about an hour to finish.
## Preparing Nanopore reads for Assembly
--------------
The title is a bit of a misnomer, we will actually be predicting the size of the assembly using a program called [Jellyfish](https://github.com/gmarcais/Jellyfish) followed by [GenomeScope](https://github.com/schatzlab/genomescope). 
### [Jellyfish](https://github.com/gmarcais/Jellyfish)

Jellyfish simply counts all the kmer combinations and makes a histogram text file out of it. K-mers are just the base pair combinations with "n" number of base pairs for example.

```
a simple phrase like
"the cat in the hat"
with kmer size 3
becomes...

thecatinthehat
the
 hec
  eca
   cat
    ati
     tin
      int
       nth
        the
         heh
          eha
           hat
with kmer "the" having 2 replicates
````
To process you're reads through jellyfish simply use 
```
jellyfish count -m 16 -s 100M -t 8 -C INPUT-READS-HERE.fasta
```
* -m : the kmer size you wish to use (I usually need 16 but 21 is the "average")
* -s : hash table size (it increases as needed to better to under estimate as 100M)
* -t : threads, just match to the CPU of your node or 2 less than the CPU of your laptop (unless you dont mind overclocking)
* -C : canonical, the kmer is treated the same even when backwards.
* INPUT-READS-HERE.fasta : the name of your input read file.


**Now that the mer_counts.jf file has been generated we can now tell jellyfish to make the actual histogram file**

```
jellyfish histo -t 10 --high=1000000 /path/to/mer_counts.jf > Output-histogram-16.histo
```
* -t : threads, just match to the CPU of your node or 2 less than the CPU of your laptop (unless you dont mind overclocking)
* /path/to/mer_counts.jf , full path to wherever the mer_counts.jf file was made in the previous jellyfish command
* Output-histogram-16.histo , name of the output file
   
### [GenomeScope](https://github.com/schatzlab/genomescope)

Once you have the .histo file made by jellyfish you have two ways of having genomescope.

1. The first and the easiest way is though the browser [genomescope](http://genomescope.org/genomescope2.0/)

* Simply drag and drop your .histo file onto the webpage and change the kmer to the same used in the jellyfish count command
  
2. If the browser version is down or needs the cache cleared, their is also the [downloadable](https://github.com/schatzlab/genomescope) verison

```
genomescope2 -i input.histo -p ploidy_number -k kmer_number -n name -o output_directory/
```
------

Whichever method you use once you get the results from genomescope they will look something like this...

![GenenomeScope example output](https://github.com/Artifice120/Nanopore-Assembly-to-Annotation/assets/160672410/8050063f-40b6-4664-90b0-ddaa4c2a46ac)

*Once you have this graph note the estimated length at the top of the page for the next step

## Assembling nanopore reads using [Canu](https://github.com/marbl/canu) on a slurm cluster
-----------
### Why Canu ?
Despite having an overall longer runtime [Canu](https://github.com/marbl/canu) makes up for that by being able to continue were it left off meaning nodes on slurm with short run-times and wait times can be used in series which results in an overall smaller queue time. Also thanks to its scoring system there are less reads that are ignored and instead are given a score allowing assemblies to be produced with less read coverage needed.
### Why not Canu ?
If you already have high read coverage (~80) Canu may not be the best option as it will reach its peak in quality which can be surpassed by other assemblers that require higher coverage but produce faster and better contigs/assemblies such as Trinity.

## [Canu](https://github.com/marbl/canu)
----
### **The command line**

What sets Canu apart from other assmeblers is that it is runs very independently. By that I mean it is able to optimize its own parameters once it knows what permissions and configurations you have.
In the most simply case if you have a single computer simply use the minimal options in the command line below:
```
canu -p prefix/name -d /path/to/directory/to/output/ genomeSize=436m utgReAlign=true overlapper=mhap -nanopore /path/to/input_reads.fastq
```
* -p prefix/name, arbitrary prefix to go in front of output files
* -d output directory, path to directory where the output files and temporary files will go to. Canu will make the specified directories if they don't exist.
> * **genomeSize= , INTEGER that can be followed by mega (m) or giga (g) prefix same as predicted length in Genomescope output**
* utgReAlign= , recommended for eukaryotic sequences.
* overlapper=mhap , the default and best overlapper method, the lesser minimap ovelapper method is selected if the computer cannot handle the load or because of impatience.
* -nanopore , These are nanopore inputs after all
* /path/to/input_reads.fastq , the path to the nanopore fastq read file(s)

> * If this is being run on a slurm cluster and only 1 node is being used it is best to use **useGrid=false** with the command above since canu will maximize the resources of the one node.
> * If you are using multiple node at the same time/ in parallel on a slurm cluster use **useGrid=remote** along with the **gridOptions="--partition=campus --qos=campus --time=24:00:00 --account ACF-UTK0011"** changing the specifics to match your slurm parameters.
> * If you are lucky enough to have no limits to the number of slurm jobs you can run use **useGrid=true** & **gridOptions="--partition=campus --qos=campus --time=24:00:00 --account ACF-UTK0011"**  which will allow canu to submit its own job to the cluster and also work in parallel with other nodes.

### **Prelim Reports**

after ~ 1 hour of canu running there be a kmer histogram generated by canu like the one below

![image](https://github.com/Artifice120/Nanopore-Assembly-to-Annotation/assets/160672410/00b0c791-3693-4cca-b016-fea96cafb633)

Just make sure there is only 1 peak that is visible from the noise to the left.
If there are multiple peaks it may be a sign of significant contamination or ploidy.
If there are no peaks than the estimated genome size is probably too small or the concentration of the sample was too low when being sequenced

### **Handling repetitive sequences**

If a seqeunce is especially repetitive it will require significantly more time and storage space (20+TB) to minimize this more stringent settings can be placed on the corMhap filter with the options 

```
corMhapFilterThreshold=0.0000000002 corMhapOptions="--threshold 0.80 --num-hashes 512 --num-min-matches 3"
```

------
## [Busco](https://busco.ezlab.org/)

Overview:

After obtaining your contigs from you're assembler the program BUSCO is used to estimate the completeness and quality of your assembly. Even though the contigs are almost gaurunteed to need additional "polishing" done; The Busco results are still useful for determining in what ways the assembly needs to be improved or re-assembled.

> Installation Tip: Had some issues with installing into a conda environment. Noticed it did not give me any issues when I installed it into an environment with blobtoolkit and minimap already installed. 

```
busco -i /path/to/your/contigs.fasta -t thread# -l closest_lineage -m geno -o /directory/for/ouput/files/ -f
```
* -i input, this is the file path to your contigs you would like scored
* -l lineage, usually just copy and paste lineage name from their [website](https://busco.ezlab.org/list_of_lineages.html) and paste the one that is most closely related to the orgainism that is being sequenced.
> * leave out the date in the name of the lineage ( ex: hemiptera_odb10 not hemiptera_odb10.2019-04-24 )
* -m mode, assuming this is a genome set this to "geno"
* -o output, the path to the directory to deposit BUSCO's output files
* -t threads, the number of threads/CPU you would like to use. Default= 1, usually 1 thread takes ~15-30 minutes to finish
* -f force, when present overwrites any output files that already exist in the same output directory.

### Example BUSCO output

![image](https://github.com/Artifice120/Nanopore-Assembly-to-Annotation/assets/160672410/6a4d9504-3899-44cb-bd87-e79648d8de4e)

**In this example there are several issues with the assembly**
1. The fragmentation and missing busco percentage is very high (30%). At this amount the assembly has to be redone with diffrent reads or reads that are mapped/selected diffrently
2. The duplication level is somewhat high (21%). This can be significantly reduced by purging the haplotigs.
------
## [Purging haplotigs](https://bitbucket.org/mroachawri/purge_haplotigs/src/master/)

[Purge haplotigs](https://bitbucket.org/mroachawri/purge_haplotigs/src/master/)  is useful for assesing the quality of the reads with a contig coverage histogram similar to genomescope as well as reducing duplication with the purge function.

### Before the histogram can be formed or haplotigs can be purged, the reads used for assembly must first be aligned/mapped to your contigs/assembly in question.
```
minimap2 -t 48 -ax map-ont /path/to/contigs.fasta /path/to/reads.fa --secondary=no | samtools sort -m 10G -o /path/to/aligned.bam -T /path/to/tmp.ali
```
* -t threads, the number of threads/CPU's to be used by this program. (48 threads on a 48 CPU node worked)
* -ax preset options, map-ont is the default and is what worked...
* /path/to/contigs.fasta , the path to the fasta contig file being polished
* /path/to/reads.fa , the path to the reads used in the assembly of the contigs.
> * This program can only handle [sanatized reads](https://github.com/Artifice120/Nanopore-Assembly-to-Annotation/tree/main?tab=readme-ov-file#rough-draft-for-a-guide-on-one-way-to-assembly-and-annotate-nanopore-reads) otherwise it will just crash with many error messages.
* --secondary=no , specifying that secondary alignment output is not needed.
------
* -m memory, the amount of memory(RAM) to allocate to samtools (check memory of node/machine to set value)
* -o /path/to/aligned.bam , path to where you want the output, it must be named aligned.bam
* -T /path/to/tmp.ali , path for temporary file(s), this actually does need to be named tmp.ali

**Once finished there should be a aligned.bam file**

### Generating histogram with PurgeHaplotigs

After generating a aligned.bam file the coverage histogram can be generated with the command below:
```
purge_haplotigs  hist  -b /path/to/aligned.bam -g /path/to/contigs.fasta -t 30
```
* -b /path/to/aligned.bam , path to aligned.bam file
* -g /path/to/contigs.fasta , path to the same contigs in question.
* -t threads, number of threads/CPU to allocate to this task (Check CPU's that are available on your machine or node)

### Using the histogram

Here is an example of a read coverage histogram


 ![Contig_Polishing histogram 200](https://github.com/Artifice120/Nanopore-Assembly-to-Annotation/assets/160672410/f45d4154-ac43-44b9-8984-4deb24844100)

This histogram is used to determine the; -l low , -m mid , and -h high points to be used in the following coverage analysis step.

I found it useful to go through the [issue page of purge haplotigs](https://bitbucket.org/mroachawri/purge_haplotigs/issues?q=histogram) in the closed issue section to get a feel for what selections worked and did not work for a given histogram.
In this case I chose -l 8  -m 55  -h 140 since the points between -m and -h went through more stringent filtering than the contigs between points -l and -m.

### Read Coverage Analysis

Once the -l -m and -h values have been determined from the histogram adn the aligned.bam.200.gencov file has been located the following command can be used to obtain the coverage stats csv file that will allow the correct sections to be purged from the contigs.

```
purge_haplotigs  cov  -i /path/to/aligned.bam.200.gencov -l #  -m #  -h #  -o coverage_stats.csv -j 80  -s 80
```
* -i /path/to/aligned.bam.200.gencov , input, the path to the aligned.bam.200.gencov file generated from the histogram step.
* -l #  -m #  -h # , the -l low -m mid -h high values to be used based on the histogram peak(s).
* -o output, name of the output file (coverage_stats.csv is convienent since the pipline is not finished yet)
* -j junk, label a contig as "junk" if this percentage of it is below the -l cutoff or above the -h cutoff. (default=80)
* -s suspect, label as "suspected haplotig" if this percentage or less of a contig is diploid level converage.

### Actually Purging the Haplotigs

Once the coverage stats csv file has been generated the purging command can be used to generate the final curated contig files.
```
purge_haplotigs  purge  -g /path/to/same/contigs.fasta  -c coverage_stats.csv -t 30
```
* -g input, place the file path to the same contig fasta file that you have been using to this point.
* -c coverage, plave the path to the coverage_stats.csv file made in the previous step.
* -t threads, number of threads/CPU's you wish to use (check the CPU's available for your machine or node)

**Once the curated fasta files are generated feel free to run the new file through busco to see how the scores compare to the original contigs**

> Poof, your a wizard!
 
------
## [Blobtools](https://github.com/blobtoolkit/blobtoolkit)

Now that there are contig files with a satisfactory Busco score, there needs to be a way to visualize all the information (Busco, N50, coverage, taxonomy, GC content ...)
[Blobtools](https://github.com/blobtoolkit/blobtoolkit) orgainizes this information into a human readable figures that can manipulated with a guided user interface.

# Installing Blobtools

Since Blobtools is originally made to be installed with python (ie, pip). A python virtual environment is easier to use.

Create a local virtual environment with the command below;

```
python -m venv path/you/want/env/with/name
```
This will make a "empty" directory for your virtual environment at the specified path.

Once the virtual environment is made activate it with the following command structure :

```
source path/to/env/you/just/made/bin/activate
```

The path will be the same path that you just made your environment in plus bin and activate.

Once activated make a conda environment in the same directory with the following command:

```
conda create -p path/to/env/name
```
Activate this environment as well.

```
conda activate path/to/env/name
```
with both environments activated the conda package required for blobtools can be installed.

```
conda install -c conda-forge firefox geckodriver
```
>Blobtools is now installed

# Using Blobtools

Once blobtools is installed a blob directory (BlobDir) can be created from your curated fasta file.
```
blobtools create --fasta /path/to/curated.fasta Name
```
* --fasta /path/to/curated.fasta , fasta option with the path to the curated fasta file.
* Name , the name of the directory to be created (It will apear on the graphs so **choose wisely**)

> Congrats its a ... blank directory?

Now that there is a blob directory we can now add info about our contigs to it.

This command will add the read coverage we already made in the purge haplotigs portion of the process.

```
blobtools add --cov /path/to/aligned.bam Name
```
* --cov /path/to/aligned.bam , coverage option with path to aligned.bam file that was made previously in the purge haplotigs prep.
* Name , name of blob directory you just made.

This next command will add the busco results for the curated reads that we also already have (happy accidents)
> * 3
>> * 2
>>> * 1
```
blobtools add --busco /path/to/busco/full_table.tsv Name
```
* --busco /path/to/busco/full_table.tsv , busco option with the path to the busco output file full_table.tsv
* Name , still the name of the blob directory you just made.
-----
Unlike the last three files added this next file has not been generated in a previous step

### **Adding taxonomy from BLAST hits**
For most large assemblies the only way to generate blast hits is through

### Installing [Blast+](https://anaconda.org/bioconda/blast) with the nr and taxonomy database.

After installing blast+ command line tool you will need to also install the non-redundant nucleotide database.
This will use ~60Gb of storage so pick a place that has room. This will take ~6 hours.
```
update_blastdb.pl --decompress nt
```
Next the taxonomy database will need to be installed specifically for blobtools

```
mkdir -p taxdump;
cd taxdump;
curl -L https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump.tar.gz | tar xzf -;
cd ..;
```
once downloaded the curated contigs can be Blast searched with.

```
blastn -db nt \
       -query /path/to/curated.fasta \
       -outfmt "6 qseqid staxids bitscore std" \
       -max_target_seqs 10 \
       -max_hsps 1 \
       -evalue 1e-25 \
       -num_threads 48 \
       -out /path/to/blast.out
```

* -db database, this option specifies what database is being used (nt)
* -query /path/to/curated.fasta , input option with path path to curated fasta file to be searched
* -outfmt "6 qseqid staxids bitscore" , must be in this format for blobtools, only prints the query sequence ID , taxonomy id number, bitscore.
*  -max_target_seqs 10 \
       -max_hsps 1 \
       -evalue 1e-25 \  , Just the default values may be changed to be more or less strict.
* -num_threads , threads/cpus to be used for the search (check CPU capability of your machine or node before setting)
* -out /path/to/blast.out , output option with path to the desired name of your .out file.

> **Note:** the backwards slash " \ " is just to have one command on multpile lines without it being treated as multiple commands (easier to read). Also known as "escaping".

now that the contigs have been blast searched with taxonomy ID's the results can be uploaded into your blob directory


```
blobtools add --replace --hits /path/to/blast.out --taxdump /path/to/taxdump/Directory/
```
* --hits /path/to/blast.out , include path to the blast.out file made in the previous step with the --hits option in front.
* --taxdump /path/to/taxdump/Directory/ , include path to the **Directory** with the --taxdump option in front.

### Viewing Blobtools Figures

To view the blobtools figures with the uploaded data you will need a computer with a gui.
This is usually not an issue unless you are using a remote cluster, In this case you will have to make sure the following command is run in an enviroment/node that support the firefox or chrome browser.

To link the browser to blobtools preform the following command beforehand while inside the conda enviroment you installed blobtools into.
```
conda install -c conda-forge firefox geckodriver
```
> Only mentioning this since it is not required for installation. Refer to their [git-hub](https://github.com/blobtoolkit/blobtoolkit#installing) if there are still issues.

To see if this is set up correctly run this command
```
blobtools view --local _
```
This will give a link to the locally hosted webpage for the blobtools viewer.

Here is an example of one such figure.

![image](https://github.com/Artifice120/Nanopore-Assembly-to-Annotation/assets/160672410/f9af0946-f697-4d88-a0e3-848b5365803c)

In this image it is possibly to see that despite having a high Busco score; there is significant Pseudomonadota contamination in the assembly.

Exporting the corresponding table allows a all the contigs to be conveniently sorted by their taxonomy hits. Contigs that completely match the contaminating Pseudomonadot can be removed by copy and pasting a list of the "bad" contigs into a ids.txt file. then the following command can delete all contigs from the curated fasta file with ID's that match the ones in the provided in the list.

```
awk 'BEGIN{while((getline<"ids.txt")>0)l[">"$1]=1}/^>/{f=!l[$1]}f' /path/to/curated.fasta > newfile.fasta
```
* /path/to/curated.fasta , the path to the fasta file that you are "removing" contigs from.
* "ids.txt" , this is the file where all the contig ID's of the contigs to be removed needs to be manually created beforehand.  

> Some contigs may have a mixture of taxonomy matches in contigs with your target orgainism, this can be either from chimeric reads that are generated from contamination or from horizontal gene transfer or something else...

After deleting contigs another Blob Directory can be made with the new contig file and all of the other same files.
* Be sure to make a new Busco report after removing contigs to see if it is worth the loss in completeness if any.

-----

## **Annotation with [AnnotaPipeline](https://github.com/bioinformatics-ufsc/AnnotaPipeline/blob/v1.0/config/config_example.yaml)**

> The Annota Pipline is the program that worked for me at the time, if you are not sequencing insects or fungus, try NCBIs [Prokaryote](https://github.com/ncbi/pgap#pgap) or the alpha of the [Eukaryote](https://github.com/ncbi/egapx#eukaryotic-genome-annotation-pipeline---external-egapx) annotation tools.

### **Overview of [AnnotaPipeline](https://github.com/bioinformatics-ufsc/AnnotaPipeline/blob/v1.0/config/config_example.yaml)**

Annota pipline uses Augustus to generate a GFF file from an genomome assembly which is why it either requires an available refrence species or a training file for Augustus. The GFF file is then Annotated by searching through three databases one of which is your choice; However, Blast's nr database is the easiest option since BLAST+ was downloaded for the Blobtools taxonomy upload. Optional features such as Kallisto and Comet if the user also has RNA-seq and proteomic spectrometry data. 

**Notes on Installation of [AnnotaPipeline](https://github.com/bioinformatics-ufsc/AnnotaPipeline/blob/v1.0/config/config_example.yaml)**

* If you are installing using conda be sure to also download the example [configuration file](https://github.com/bioinformatics-ufsc/AnnotaPipeline/blob/v1.0/config/config_example.yaml)
* Since the Blobtools program already required the install of blast+; The databases Blast nr, Pfam, and Swissprot seems like the easiest database options.
* Once the databases are installed be sure to change the fields in the configuration file to match the locations where the databases were installed as well as the paths to python and perl.

> * Before running AnnnotaPipline be sure to check that the individual programs and corresponding databases are functional

Since all of the parameters will be entered in the Config file the actual command line is simple 

```
AnnotaPipeline.py -c /path/to/config.yaml -s /path/to/curated.fasta
```
* -c /path/to/config.yaml , config option with path to config.yaml (download example config file [here](https://github.com/bioinformatics-ufsc/AnnotaPipeline/blob/v1.0/config/config_example.yaml)
* -s /path/to/curated.fasta , input option with path to curated fasta file or gff file if you already have one somehow.

> This needs several days of continous runtime to complete all of its searches

# Annotations with E-gapx

> WARNING THIS MAY NOT WORK AS THE PROGRAM IS IN ITS ALPHA-STAGE

## Installing E-gapx

First download the git-hub files with the following commands;

```
git clone https://github.com/ncbi/egapx.git
cd egapx
```
Once downloaded, install the required dependencies into a python virtual environment in the same directory ;

```
python -m venv v-env-egap
source v-env-egap/bin/activate
pip install -r ui/requirements.txt
```
Once the dependencies are installed, install singularity in a conda environment in the same directory.
```
conda create -p c-env-egap
conda activate c-env-egap/
conda install -c conda-forge singularity
```
E-gapx can now run without root permissions while both the conda and python environment are activated

Run the following command to ensure the program is functional

```
python3 ui/egapx.py ./examples/input_D_farinae_small.yaml -e singularity -w temporary_files -o example_out
```

