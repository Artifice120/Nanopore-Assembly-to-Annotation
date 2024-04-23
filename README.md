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

### **Handling repetitive sequences**

If a seqeunce is especially repetitive it will require significantly more time and storage space (10+TB) to minimize this more stringent setting can be placed on the corMhap filter with the options 

```
corMhapFilterThreshold=0.0000000002 corMhapOptions="--threshold 0.80 --num-hashes 512 --num-min-matches 3"
```

------
## Purging




