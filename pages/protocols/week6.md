# Week 6: Binning genomes with anvi'o

## Intro (Rika will go over this at the beginning of lab)
This week in lab we’ll learn how disentangle individual microbial genomes from your mess of metagenomic contigs. We aren’t going to use a toy dataset this week—-we’re going straight into analysis with your metagenome datasets for your projects.

Genomes are disentangled from metagenomes by clustering reads together according to two properties: **coverage** and **tetranucleotide** frequency. Basically, if contigs have similar coverage patterns between datasets, they are clustered together; and if contigs have similar kmers, they will cluster together. When we cluster contigs together like this, we get a collection of contigs that are thought to represent a reconstruction of a genome from your metagenomic sample. We call these 'genome bins,' or 'metagenome-asembled genomes' (MAGs).

There is a lot of discussion in the field about which software packages are the best for making these genome bins. And of course, the one you choose will depend a lot on your dataset, what you’re trying to accomplish, and personal preference. I chose anvi’o because it is a nice visualization tool that builds in many handy features.

I am drawing a lot of information for this tutorial from the anvi'o website. If you'd like to learn more, see the link below.

http://merenlab.org/2016/06/22/anvio-tutorial-v2/

## Preparing your contigs database for anvi'o
#### 1. ssh tunnel
Boot your computer as a Mac and use the Terminal to ssh in to baross. Anvio requires visualization through the server, so this week we have to create what is called an "ssh tunnel" to log into the server in a specific way. Substitute "username" below with your own Carleton login name.

NOTE: Each of you will be assigned a different port number (i.e. 8080, 8081, 8082, etc.). Substitute that number for the one shown below.

```
ssh -L 8080:localhost:8080 username@baross.its.carleton.edu
```

#### 2. Make new folder
Make a new directory called “anvio” inside your project folder, then change into that directory.

```
cd project_directory
mkdir anvio
cd anvio
```

#### 3. Copying data

Last week, we learned how to make BAM files. This week, we'll need them to make our bins, because they give us coverage information. You need BAM files that show the coverage of your own reads mapped against your own contigs, as well as other BAM files showing other people's reads mapped against your contigs.

Fortunately for you, I made all of these BAM files for you ahead of time. They are in this folder:

/workspace/data/Genomics_Bioinformatics_shared/Tara_mappings

Choose **three** mapping files from that folder. The ones you choose should all be from your own dataset. The names are like this, as an example:

`ERR599021_assembled_vs_ERR599013_reads_sorted.bam`

Where ERR599021 is the reference, and reads from ERR599013 were mapped to it. You want only the bam files with your **own** sample number as the **reference**.

You should also include the bam file of your own reads mapped against your own contig.

```
cp [path to sorted .bam files] .
cp [path to sorted .bai files] .
```

For example:
```
cp /workspace/data/Genomics_Bioinformatics_shared/Tara_mappings/ERR599021_assembled_vs_ERR599021_reads_sorted.bam
cp /workspace/data/Genomics_Bioinformatics_shared/Tara_mappings/ERR599021_assembled_vs_ERR599021_reads_sorted.bam.bai
cp /workspace/data/Genomics_Bioinformatics_shared/Tara_mappings/ERR599021_assembled_vs_ERR599013_reads_sorted.bam
cp /workspace/data/Genomics_Bioinformatics_shared/Tara_mappings/ERR599021_assembled_vs_ERR599013_reads_sorted.bam.bai
cp /workspace/data/Genomics_Bioinformatics_shared/Tara_mappings/ERR599021_assembled_vs_ERR599075_reads_sorted.bam
cp /workspace/data/Genomics_Bioinformatics_shared/Tara_mappings/ERR599021_assembled_vs_ERR599075_reads_sorted.bam.bai
```

Finally, copy your contigs over as well. They are probably in your assembly directory. For example:
```
cp ../ERR598983_assembly/ERR598983_assembled_reformatted.fasta .
```

#### 4. Get gene calls and annotations from Prokka
The first thing you have to do is make contigs database, which contains the sequences of your contigs, plus lots of information about those contigs. This includes information from Prokka-- so we have to take the information from Prokka and put it in our contigs database.

First, you have to run a script on your Prokka files to convert them into a text file that we can import into anvi'o. Navigate to wherever your Prokka results are for your project assembly, and run a script to extract information from that file. Then copy it to your anvi'o folder and change directory to your anvio folder. For example:

```
cd prokka_project
gff_parser.py PROKKA_10072018.gff --gene-calls gene_calls.txt --annotation gene_annot.txt
cp gene_calls.txt ../../anvio
cp gene_annot.txt ../../anvio
cd ../../anvio
```


#### 5. Make the contigs database

Now, you make the contigs database. It will have lots of information about-- wait for it-- your contigs!

-`anvi-gen-contigs-database` is the anvi’o script that makes the contigs database.

-`–f` is the fasta file with your contigs that you have already assembled and fixed.

-`–o` provides the name of your new contigs database.

-`external_gene_calls` provides the name of the Prokka file you just made so you can import the Prokka calls into your contigs database

-`--ignore-internal-stop-codons` will ignore any internal stop codons in your gene calls. Sometimes these will get included in your Prokka results by accident, but for our purposes we can ignore them.

```
anvi-gen-contigs-database -f [your formatted, assembled contigs] -o contigs.db --external-gene-calls gene_calls.txt --ignore-internal-stop-codons
```

#### 6. Import the Prokka annotations
Import your prokka results like this:
 ```
 anvi-import-functions -c contigs.db -i gene_annot.txt
 ```

#### 7. Search for single copy universal genes
Now we will search our contigs for archaeal and bacterial single-copy core genes. This will be useful later on because when we try to disentangle genomes from this metagenome, these single-copy core genes can be good markers for how complete your genome is.

This process is slow, so we're going to run it on 5 CPUs rather than just 1. You can run it on screen in the background while you move forward with step 6. It should take a little under 10 minutes.

```
screen
anvi-run-hmms -c contigs.db -T 5
```

#### 8. Determine taxonomy using Centrifuge
Now we are going to figure out the taxonomy of our contigs using a program called centrifuge. Centrifuge is a program that compares your contigs to a sequence database in order to assign taxonomy to different sequences within your metagenome. We're going to use it first to classify your contigs.

If you would like to know more, go here: http://merenlab.org/2016/06/22/anvio-tutorial-v2/ and here: http://www.ccb.jhu.edu/software/centrifuge/

First, export your genes from anvi'o.
```
anvi-get-dna-sequences-for-gene-calls -c contigs.db -o gene-calls.fa
```

#### 9. Run Centrifuge
```
centrifuge -f -x /usr/local/CENTRIFUGE/p_compressed gene-calls.fa -S centrifuge_hits.tsv
```

#### 10. Import Centrifuge data
Now import those centrifuge results for your contigs back in to anvi'o. Anvi'o can automatically read and import centrifuge output.
```
anvi-import-taxonomy -c contigs.db -i centrifuge_report.tsv centrifuge_hits.tsv -p centrifuge
```

## Incorporating mapping data

We've just decorated our contigs database with all kinds of things. Now, we incorporate all of the BAM files.

#### 11. Import mapping files into anvi'o with anvi-profile
Now anvi’o needs to combine all of this information (your mapping, your contigs, your open reading frames, and your taxonomy) together. To do this, use anvi-profile.

-`anvi-profile` is the name of the program that combines the info together
-The `–i` flag provides the name of the sorted bam file that you copied in the step above.
-The `-T` flag sets the number of CPUs. There are 11 of you, and 96 to spare. For now, let's set it to 5 so we don't blow up the server.
-The `-M` flag sets a minimum contig length. In a project for publication, you'd want to use at least 1000, because the clustering of contigs is dependent on calculating their tetranucleotide frequencies (searching for patterns of kmers). You need to have a long enough contig to calculate these frequencies accurately. But for our purposes, let's use 500 so you can use as many contigs as possible.
```
anvi-profile -i [your sorted bam file] -c contigs.db -T 5 -M 500
```

#### 12. Merge them together with anvi-merge
Now merge all of these profiles together using a program called anvi-merge. You have to merge together files in directories that were created by the previous profiling step. The asterisk * is a wildcard that tells the computer, 'take all of the folders called 'PROFILE.db' from all of the directories and merge them together.'

We're also going to tell the computer not to bin these contigs automatically (called 'unsupervised' binning), we want to bin them by hand ('supervised' binning). So we use the --skip-concoct-binning flag.

This step will take a couple minutes.
```
anvi-merge */PROFILE.db -o SAMPLES-MERGED -c contigs.db --skip-concoct-binning
```
## Visualizing and making your bins

#### 13. anvi-interactive
Now the fun part with pretty pictures! Type this to open up the visualization of your contigs (of course, change the port number to the one you were assigned):
```
anvi-interactive -p SAMPLES-MERGED/PROFILE.db -c contigs.db -P 8080
````

#### 14. Visualize in browser
Now, open up a browser (Chrome works well) and type this into the browser window to stream the results directly from the server to your browser. NOTE that each of you will be assigned a different port number (i.e. 8080, 8081, 8082, etc); use the one you logged in with in the first step.

http://localhost:8080

Cool, eh?

Click 'Draw' to see your results! You should see something like this:
![anvio screenshot](../images/anvio.png)

What you are looking at:

-the tree on the inside shows the clustering of your contigs. Contigs that were more similar according to k-mer frequency and coverage clustered together.

-the rings on the outside show your samples. Each ring is a different sample. This is a visualization of the mapping that you did last week, but now we can see the mapping across the whole sample, and for all samples at once. There is one black line per contig. The taller the black line, the more mapping for that contig.

-the 'taxonomy' ring shows the centrifuge designation for the taxonomy of that particular contig.

-the 'GC content' ring shows the average percent of bases that were G or C as opposed to A or T for that contig.

#### 15. Make bins
We will go over the process for making bins together in class.

Because your datasets are fairly small, your bins are also going to be very small. Your percent completeness will be very low. Try to identify ~3-5 bins according to patterns in the mapping of the datasets as well as the GC content.

When you are done making your bins, be sure to click on 'Store bin collection', give it a name ('my_bins' works), and then click on 'Generate a static summary page.' Click on the link it gives you. It will provide lots of information about your bins. In the boxes under the heading 'taxonomy,' you can click on the box to get a percentage rundown of how many contigs in your bin matched specific taxa according to centrifuge, if any matched.

**Once you have completed your binning process, take a screenshot of your anvi'o visualization and save it as 'Figure 1.' Write a figure caption explaining what your project dataset is, and which datasets you mapped to your sample.**

#### 16. Finding bin information
You will find your new bin FASTA files in the directory called '~/project_directory/anvio/SAMPLES-MERGED/SUMMARY_my_bins'.

`bins_summary.txt` provides just that, with information about the taxonomy, total length, number of contigs, N50, GC content, percent complete, and percent redundancy of each of your bins. This is reflected in the summary html page you generated earlier when you clicked 'Generate a static summary page.'

If you go to the directory `bin_by_bin`, you will find a series of directories, one for each bin you made. Inside each directory is a wealth of information about each bin. This includes (among other things):

-a FASTA file containing all of the contigs that comprise your bin (i.e. `Bin_1-contigs.fa`)

-mean coverage of each bin across all of your samples (i.e. `Bin_1-mean-coverage.txt`)

-files containing copies of single-copy, universal genes found in your contigs (i.e. `Bin_1-Rinke_et_al_hmm-sequences.txt` and `Bin_1-Campbell_et_all-hmm-sequences.txt`

-information about single nucleotide variability in your bins-- the number of SNVs per kilobasepair. (i.e. `Bin_1-variability.txt`)

## Analyzing your bins
For this week's post-lab assignment, you won't do a mini-research question because the quality of your bins may vary, through no fault of your own. (That's real data for you...) So instead, we're going to answer a few questions about the data that you just generated.

Most of these questions can be answered with the static summary page that you can visualize in your web browser. However, you can also look at the text files that anvi'o generates-- I tell you which files to look at below.


##### 1. **Bin completeness**

The easiest bins to generate are often the ones that had the longest N50 and the lowest population diversity. Take a look at this file:

`SAMPLES-MERGED/SUMMARY_my_bins/bins_summary.txt`.

1a. Which bin was the most complete?

1b. What was the N50 of that bin?

1c. What was the anvi'o-labeled taxonomy of this bin?




##### 2. **Coverage across all bins for one sample**

By comparing different bins in a single sample, we can get a sense of which microbes are the most abundant in one sample. We could then later take a look at what genes are in those microbes to give us a sense of why they're so abundant.


Use the `SAMPLES-MERGED/SUMMARY_my_bins/bins_across_samples/mean_coverage.txt` file to answer these questions.

2a. Which bin had the highest coverage, when looking only at the mapping of your own dataset against itself?

2b. What is the taxonomy of that bin, according to anvi'o?

2c. Make a bar graph in which each bin is a different bar, and the bar height indicates the mean coverage. Call this 'Figure 2' and include a figure caption.

2d. What does it mean, biologically/ecologically speaking, for a bin to have high coverage?



##### 3. **Coverage of one bin across samples**

By comparing one bin across many samples, we can get a sense of where certain microbes grow the best. Again, we could then later take a look at what genes are in those microbes to give us a sense of why they're so abundant.


Choose one bin that you're interested in. Use the file `SAMPLES-MERGED/SUMMARY_my_bins/bin_by_bin/Bin_X/Bin_X-mean_coverage.txt` to answer these questions.

3a. Make a bar graph in which each sample is a different bar, and the bar height indicates the coverage of your bin in that sample. Call this "Figure 3" and include a figure caption.

3b. In which sample does your bin have the highest coverage? Second highest? What might this imply about the abundance of this microbe in different ecosystems?




##### 4. **Variability of all bins for one sample**

anvi'o can identify SNVs in your BAM files, and it can tell you all about the SNVs in your bin.
Use the `SAMPLES-MERGED/SUMMARY_my_bins/bins_across_samples/variability.txt` file to answer these questions.

4a. Which bins had the highest and lowest variability (single nucleotide variants per kilobase pair (SNVs/kbp) across all samples, when looking only at the mapping of your own dataset (self-to-self)?

4b. What is the taxonomy of those bins, according to anvi'o?

4c. SNVs are usually removed from a population after some sort of selective sweep (like an extinction event) or when one microbe starts to reproduce very quickly (like an algal bloom). SNVs start to appear if there has been enough time for mutations to build up in the population. Based on your bins, what does the SNV information tell you about the microbial populations in your specific sample? (Keep in mind that this information tells you about the population of closely related organisms to your bin, not just one specific individual cell.)

*Important caveat! If your sample had low coverage (i.e. less than 10), that may be skewing the variability results because if you don't have any reads mapping, there are no SNVs to count!*

**Compile Figures 1-3 and your answers to questions 1-4 and submit on the Moodle by lab time next week.**

#### Final step: sharing data
Some of you might want access to each others' anvio data for your final projects. So let's share it on in our shared folder.

First, make a directory with your name on it, and then put your contigs database and SAMPLES_MERGED directory in there.

```
mkdir /Accounts/Genomics_Bioinformatics_shared/anvio_stuff/[your name]
cp contigs.db /Accounts/Genomics_Bioinformatics_shared/anvio_stuff/[your name]
cp -r SAMPLES_MERGED /Accounts/Genomics_Bioinformatics_shared/anvio_stuff/[your name]
```
