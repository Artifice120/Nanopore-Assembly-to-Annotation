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

This would make a histogram with a large amount of kmers in the 1 repeat bin and one at 2 repeats histogram

kmers with low amounts of repeats are considered "noise" as sequencing usually ensures each part of a seqeunce is recorded several times known as "read coverage"
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
If you already have high read coverage (~40 to 80) Canu may not be the best option as it will reach its peak in quality which can be surpassed by other assemblers that require higher coverage but produce faster and better contigs/assemblies such as NextDenovo.

Canu is also only able to use either pacbio or nanopore long reads. If you have illumina short reads and/or Hi-C reads HiFiAsm may be a better option since it is able to incorpate all those read types at once during assembly.

## [Canu](https://github.com/marbl/canu)
----
### **The command line**

What sets Canu apart from other assmeblers is that it is runs very independently. By that I mean it is able to optimize its own parameters once it knows what permissions and configurations you have.
In the most simple case if you have a single computer use the minimal options in the command line below:
```
canu -p prefix/name -d /path/to/directory/to/output/ genomeSize=436m utgReAlign=true overlapper=mhap -nanopore /path/to/input_reads.fastq
```
* -p prefix/name, arbitrary prefix to go in front of output files ( pick a name, no spaces )
* -d output directory, path to directory where the output files and temporary files will go to. Canu will make the specified directories if they don't exist.
> * **genomeSize= , INTEGER that can be followed by mega (m) or giga (g) prefix same as predicted length in Genomescope output**
* utgReAlign= , recommended for eukaryotic sequences.
* overlapper=mhap , the default and best overlapper method, the lesser minimap ovelapper method is selected if the computer cannot handle the load or because of impatience.
* -nanopore , These are nanopore inputs after all
* /path/to/input_reads.fastq , the path to the nanopore fastq read file(s)

> * If this is being run on a slurm cluster and only 1 node is being used it is best to use **useGrid=false** with the command above since canu will maximize the resources of that one node.
> * If you are using multiple node at the same time/ in parallel on a slurm cluster use **useGrid=remote** along with the **gridOptions="--partition=campus --qos=campus --time=24:00:00 --account ACF-UTK0011"** changing the specifics to match your slurm parameters. Keep in mind the initial job you create will only need 1 cpu since it is just used to submit its own canu jobs to the cluster. 
> * If you are lucky enough to have no limits to the number of slurm jobs you can run use **useGrid=true** & **gridOptions="--partition=campus --qos=campus --time=24:00:00 --account ACF-UTK0011"**  which will allow canu to submit its own job to the cluster and also work in parallel with other nodes.

### **Prelim Reports**

after ~ 1 hour of canu running there be a kmer histogram generated by canu like the one below

![image](https://github.com/Artifice120/Nanopore-Assembly-to-Annotation/assets/160672410/00b0c791-3693-4cca-b016-fea96cafb633)

Just make sure there is only 1 peak that is visible from the noise to the left.
If there are multiple peaks it may be a sign of significant contamination or ploidy.
If there are no peaks than the estimated genome size is probably too small or the concentration of the sample was too low when being sequenced
> If you want to learn the ploidy of your organism try using [smudge-plot](https://github.com/KamilSJaron/smudgeplot)

### **Handling repetitive sequences**

If a seqeunce is especially repetitive it will require significantly more time and storage space (20+TB) to minimize this, more stringent settings can be placed on the corMhap filter with the options 

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

However, python 3.9.21 is recommended. in order to install without root permissions if anaconda or micromamba is already installed you can create a conda environment with this exact verion:

```
micromamba create -n python3.9-env
micromamba activate python3.9-env
micromamba install -c anaconda -c conda-forge python==3.9.21
```
Now that a conda environment with the correct version of Python is activated

Create a local virtual environment with the command below;

```
python3.9 -m venv path/you/want/env/with/name
source path/you/want/env/with/name/bin/activate
```
Once the actual virtual evironment is built and activated the blobtools dependencies and executables can be intalled with

```
pip install blobtoolkit[full]
```
This will be missing the blobtoolkit-api and blobtoolkit-viewer exectables and will need to be added to the environment manually

```
cd path/you/want/env/with/name/bin/
wget https://github.com/genomehubs/blobtoolkit/releases/download/4.4.5/blobtoolkit-api-linux
wget https://github.com/genomehubs/blobtoolkit/releases/download/4.4.5/blobtoolkit-viewer-linux
chmod 777 *
mv blobtoolkit-api-linux blobtoolkit-api
mv blobtoolkit-viewer-linux blobtoolkit-viewer
```
Also to limit issues between conda and python environments also need to sym-link the python3.9 executable to the environment

```
micromamba activate python3.9-env
cd path/you/want/env/with/name/bin/
ln -s $(which python3.9) python3.9
```
To activate blobtools 
```
source path/you/want/env/with/name/bin/activate

blobtools
```
If the command line options appeared it is installed

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
cd ..
```
Once downloaded the curated contigs can be Blast searched with.

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

# **Annotation with [Helixer](https://github.com/weberlab-hhu/Helixer)**
> An AI gene prediction tool with pretrained models spanning fungi, vertibrates, invertebrates, and land plants.
> Is much more complete and accurate compared to E-gapx and Braker at least for aphids in my experience.

### **Docker installation of [Helixer](https://github.com/weberlab-hhu/Helixer)**

The most hassle free approach to installing Helixer is the docker image using singularity (I don't have root access).

```
singularity pull docker://gglyptodon/helixer-docker:helixer_v0.3.4_cuda_12.2.2-cudnn8
```
Helixer and all of its depencies will be installed.

The models that Helixer uses need to be downloaded next with the following commands.

```
cd /path/to/save/location
singularity -B /path/to/save/location exec /path/to/container/helixer-docker.sif fetch_helixer_models.py -l [ pick "fungi" , "invertebrate" , "vertibrate" or "land_plant" ]
```
* /path/to/save/location : The directory that the model files will save to ( best to save them to an empty folder/location ). Be sure to include them in the both next to the "cd" and "-B"
* /path/to/container/helixer-docker.sif : The directory to where the helixer container was saved to in the previous step
* -l : specify the lineage that you want to download. The choices "fungi" , "invertebrate" , "vertibrate" and "land_plant" are available. If you want to download all of them just type "-a" instead.

To run the helixer singularity container within an sbatch or bash script use following format:
```
singularity exec --nv --env 'RUST_BACKTRACE=full' /path/to/container/helixer-docker_helixer_v0.3.3_cuda_11.8.0-cudnn8.sif Helixer.py
```
* singularity exec : Runs/executes a single command within the sinngularity container. Think of it as a smaller and seperate computer within this one.
* --nv --env 'RUST_BACKTRACE=full' : ensures the rust dependency is operational
* path/to/container/helixer-docker_helixer_v0.3.3_cuda_11.8.0-cudnn8.sif : this is the file location of wherever you saved the singularity container.
* Helixer.py : the actual command you want to run (Example in this case)

> Now that the singularity portion is explained, th full helixer command is as follows:

```
singularity exec --nv --env 'RUST_BACKTRACE=full' /lustre/isaac/scratch/jtorre28/singularity_containers/helixer-docker_helixer_v0.3.3_cuda_11.8.0-cudnn8.sif Helixer.py --fasta-path /lustre/isaac/scratch/jtorre28/next-denovo/NextDenovo/ue/ue-nd-patched.filtered.fa --gff-output-path /lustre/isaac/scratch/jtorre28/AI-ue-reader/ue-nd-patched.gff --lineage invertebrate --species Uroleucon_erigeronense --lineage /lustre/isaac/scratch/jtorre28/singularity_containers/tmp
```

* --fasta-path : is followed by the file path for the fasta file with the assembled and polished contigs that are to be annotated.
* --gff-output-path : is followed by the anticipated file location to output the gff file (annotations). NOTE: Although the file will be created, any directories that dont exist will not be created resulting in the output file being placed in the working directory. Make the directory before running helixer if needed.
* --lineage : this lineage refrences to the model that will be used. Make sure you already have the correspoding models downloaded before running.
* --species : Just pick any name to be used in the files ( it does nothing else )
* --tmp : Place to store temporary running files, only matter in a place like a slurm cluster where diffrent directories have diffrent storage and I/O limits.

# Annotations with E-gapx

> WARNING: THIS MAY NOT WORK FOR YOU'RE ASSEMBLY AS THE PROGRAM IS IN ITS BETA-STAGE

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
# Functional Annotation

By this point you should have a .gff or .gtf file that has a reasonable amount of genes.

Usually the third column will have a label with gene, cds, intron, or exon. One way to quickly count the number of genes is to count the number of times "gene" is written in the label column (usually 3)
```
awk '$3 == "gene"' path/to/file.tsv | wc -l
```
This TSV file can be funcitonally annotated with Entap 

## Installing Entap

The easiest way to install is with a singularity image since it installs all the depencies while not requiring the root 

```
singularity pull entap.sif docker://plantgenomics/entap:latest
```
After the image is installed the configuration files still need to be obtained with the following commands.

```
singularity shell /path/to/entap.sif
EnTAP --config
```
This will give you an "error" message like below
```
EnTAP config ini not found, generated at: /lustre/isaac/scratch/jtorre28/entap/entap_config.ini
EnTAP run parameter ini not found, generated at: /lustre/isaac/scratch/jtorre28/entap/entap_run.params
Error code: 14
Configuration file was not found and was generated for you, make sure to check the paths before continuing.
```
This message can be ignored since the files get genrated for you afterwards.

The actual configuration command can then be run as follows.

```
EnTAP --config --run-ini entap_run.params --entap-ini entap_config.ini
```
Unfortunantly the paths for configured database are saved inside the singularity image so it is important that you keep the log file and the debug file generated by the configuration so the database paths can be used when runing EnTap.

The path the debug file will be in the Stdout(on the screen) or in a log file if you are using Slurm and will look like this.

```
Parsing ini file at: entap_run.params
Parsing ini file at: entap_config.ini
ini files parsed, debug logging will continue at: /lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani2/debug_2024Y8M2D-10h20m22s.txt
```
in this example the debug file would in /lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani2/debug_2024Y8M2D-10h20m22s.txt

open this file to find the database paths like so 

```
rsem-sam-validator: /usr/local/bin//libs/RSEM-1.3.3//rsem-sam-validator
rsem-prepare-reference: /usr/local/bin//libs/RSEM-1.3.3//rsem-prepare-reference
convert-sam-for-rsem: /usr/local/bin//libs/RSEM-1.3.3//convert-sam-for-rsem
transdecoder-long-exe: TransDecoder.LongOrfs
transdecoder-predict-exe: TransDecoder.Predict
transdecoder-m: 100
transdecoder-no-refine-starts: false
diamond-exe: /usr/local/bin/diamond
taxon: Aulacorthum_solani
qcoverage: 50.000000
tcoverage: 50.000000
contam: bacteria,
e-value: 0.000010
uninformative: conserved,predicted,unknown,unnamed,hypothetical,putative,unidentified,uncharacterized,uncultured,uninformative,
ontology_source: 0,
eggnog-map-exe: emapper.py
eggnog-map-data: /lustre/isaac/scratch/jtorre28/entap/entap_outfiles/databases
eggnog-map-dmnd: /lustre/isaac/scratch/jtorre28/entap/entap_outfiles/databases/eggnog_proteins.dmnd
interproscan-exe: interproscan.sh
interproscan-db:
hgt-donor:
hgt-recipient:
hgt-gff:

------------------------------------------------------
EnTAP Database Configuration
------------------------------------------------------
Database written to: /lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani1//bin/entap_database.bin

------------------------------------------------------
DIAMOND Database Configuration
------------------------------------------------------
DIAMOND database skipped, exists at: /lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani1//bin/uniprot_trembl

------------------------------------------------------
EggNOG Database Configuration
------------------------------------------------------
EggNOG SQL Database skipped, exists at: /lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani1//databases/eggnog.db
EggNOG DIAMOND database skipped, exists at: /lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani1//bin/eggnog_proteins.dmnd


EnTAP has completed!
Total runtime (minutes): 3
```

In this example the Entap database is saved to /lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani1//bin/entap_database.bin

DIAMOND database is saved to /lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani1//bin/uniprot_trembl

and the EGGNOG databases are saved to /lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani1//databases/eggnog.db and /lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani1//bin/eggnog_proteins.dmnd

Once you have the paths to these three databases you need to manually input these into the file entap_config.ini so the program can actually run 

They will be input into this section:

```
#-------------------------------
# [entap]
#-------------------------------
#Path to the EnTAP binary database
#type:string
entap-db-bin=/lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani1//bin/entap_database.bin
#Path to the EnTAP SQL database (not needed if you are using the binary database)
#type:string
entap-db-sql=/usr/local/bin//databases/entap_database.db
#Path to the EnTAP graphing script (entap_graphing.py)
#type:string
entap-graph=/usr/local/bin//src/entap_graphing.py
```

> EnTap is now Configured :)

## Running EnTap

To run EnTap instead of directly typing all the options and parameters on the command line they are all listed in a parameters file "entap_run.params"

This has the benifit of being able to see all the defaults and options at once while also being able to save your parameters.

Four databases are recomended for the DIAMOND homology searches. These databasese can be formatted from fasta files ahead of time with standalone diamond. 

```
diamond makedb -in /path/to/FASTA/file --db output/file/name
```
the paths to these diamond databases can then be added to the general portion:
```
#-------------------------------
# [general]
#-------------------------------
#Specify the output directory you would like the data produced by EnTAP to be saved to.
#type:string
out-dir=/lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani3
#Select this option if you would like to overwrite files from a previous execution of EnTAP. This will DISABLE 'picking up where you left off' which enables you to continue an annotation from where you left off before. Refer to the documentation for more information.
#type:boolean (true/false)
overwrite=true
#Path to the input transcriptome file
#type:string
input=/lustre/isaac/scratch/jtorre28/e-gapx/egapx/sol_out/trans-test.fa
#Provide the paths to the databases you would like to use for either 'run' or 'configuration'.
#For running/execution:
#    - Ensure the databases selected are in a DIAMOND configured format with an extension of .dmnd
#For configuration:
#    - Ensure the databases are in a typical FASTA format
#Note: if your databases do not have the typical NCBI or UniProt header format, taxonomic  information and filtering may not be utilized. Refer to the documentation to see how to properly format any data.
#type:list (string)
database=/lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani1/bin/uniprot_trembl.dmnd
database=/lustre/isaac/scratch/jtorre28/database/cluster-nr/nr.dmnd
database=/lustre/isaac/scratch/jtorre28/database/ref-seq/invert/invertebrate-all.protein.dmnd
database=/lustre/isaac/scratch/jtorre28/database/swissprot/swissprot.dmnd
```
Once all the required fields are finished the functional annotation can run within the singularity image with the initiation and run prameter files included

```
singularity exec entap.sif EnTAP --runN --run-ini /path/to/entap_run.params --entap-ini /path/to/entap_config.ini
```
## Adding EnTAP Homology back to GFF annotations

Output of EnTAP results are unfortunantly only available as a tsv file. In order to add the final results of the functional annotation they need to be added to the GFF attributes.

This is done with the program [AGAT](https://github.com/NBISweden/AGAT) using their ```agat_sp_manage_functional_annotation.pl``` perl script 

Before this can be done the final results of EnTAP need to be parsed to be compatible with AGAT.

The AWK script below will parse the output since the EnTAP input file enties have the same names used in the GFF file ( from GFFread earlier )

```
awk -F'\t' '{print $1"\t"$2"\t"$13 }' entap_results.tsv  | tail -n +2 | awk -F "\t" '{if ( $2 == "NA" ) {print $1"\t""hypothetical protein""\t"$3 ;} else {print $1"\t"$2"\t"$3} }' | awk -F "\t" '{if ( $3 == "NA" ) {print $1"\t"$2"\t""hypothetical protein" ;} else {print $1"\t"$2"\t"$3} }' > homology-output.tsv
```
> Be sure to change the paths of the input and output tsv files

After this file is created add the following headers so the attributes actually have names and not just values 

> Any of these may be changed as preffred except for the ID header
> Must also be tab seperated (spaces might look the same so type it instead of copy pasting to be safe)

```
ID      homology        product-homology
```

Finally, the AGAT perl script can be used to add the attributes 

```
agat_sq_add_attributes_from_tsv.pl --gff Macrosiphon.gff --tsv homology-gene-info.tsv  -o test-homology.gff3
```

* --gff : Path to the gff file 
* --tsv : Path to prepared tsv file 
* -o : name of output file









