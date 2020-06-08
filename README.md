# NZ-cicada-phylogenomics-tutorial

This will be a reference for pulling out approximately 500 AHE loci from NZ cicada assemblies.



#### Step one: preparing the list of genes
The first thing we want to do is copy the full list of genes from Eric's folder into our own folder.
```
mkdir fullAHEloci
cd fullAHEloci
cp /home/CAM/egordon/fullAHEloci/Homologs/*.fasta .
```

If we look at any of these gene alignments, we will see that the sequence headers are formatted in this way:
```
>I16239_07_ZM_CO_GRE_11_POOL_Hemiptera_Cicadidae_Afzeliada_sp_1_Copy1
```
For ease in filtering them down to our query sequences later, we will need to change the sequence header so that it only has the genus name. We will use the sed command to remove everything besides the genus.

For help in writing our sed command, [RegExr](https://regexr.com/) is a great resource for testing out regular expressions. Paste the text you will be working with in the Text box and see what your regular expression selects from it. If you need help figuring out what different symbols in a regular expression do, use the [Regex Cheat Sheet](https://www.rexegg.com/regex-quickstart.html).

The first thing we need to do is remove the beginning bit up through "Cicadidae_". We also want to test all of our commands before changing anything permanently.
```
sed -r 's/>\w+Cicadidae_/>/g' T133_L1.fasta  > testfile
head testfile
```
The -r flag is needed because our regular expression uses the extended grep vocabulary symbols. Our sequence headers in the test file should now look like this:
```
>Afzeliada_sp_1_Copy1
```
Now that the beginning bit has been removed, it is now time to try removing everything after the genus name.
```
sed -r 's/_\w+//g' testfile > testfile2
head testfile2
```
The test file sequence headers should now look like this:
```
>Afzeliada
```
This will make it easier to find sequences to use as queries.


We can now use these sed commands on the entire folder using a for loop. Note the -i flag to write the changes to the input files instead of writing to a separate output file.
```
for x in *.fasta; do sed -i -r 's/>\w+Cicadidae_/>/g' $x; sed -i -r 's/_\w+//g' $x; done
```
Now every file in this folder has the sequence headers formatted the way that we want them.



#### Step 2: Preparing our query sequences
Now that our sequence headers only have the genus name, we can run a script in Eric's folder that takes a specific taxon, searches for that taxon in all the fasta files, and creates a new folder containing those fasta files that contain that taxon with all other taxa removed. Before we run the script, we need to create a new file called taxontokeep containing the name of the taxon of interest. Since we are investigating New Zealand cicadas, we will use the genus Kikihia as it is the only New Zealand cicada in this dataset and it is also the only cicada most closely related to Maoricicada in this dataset.
```
nano taxontokeep
Kikihia
```

We can now run the Removetaxa script. It takes the folder with the AHE fasta files, a -a option, and the file taxontokeep which contains "Kikihia".
```
python /home/CAM/egordon/scripts/removeTaxa.py . -a taxontokeep
```
The script will produce a new rmtaxaout folder containing only the files that contain Kikihia and with all other taxa removed. We now need to check to see if there were other files that did not contain Kikihia and therefore are not in the rmtaxaout folder. We will do this by comparing the list of files in the rmtaxaout folder with the list of files in the fullAHEloci folder. Making sure that you are in the fullAHEloci folder, do the following:
```
ls rmtaxaout > Kfiles
ls T133* > allfiles
sort Kfiles > sortedKfiles
sort allfiles > sortedallfiles
comm -23 sortedallfiles sortedKfiles > nonKfiles
```
Our list of all files that do not contain Kikihia is in nonKfiles. We will now loop over that file and move all the listed files into the rmtaxaout folder. This will give us a folder that has all the files, but the files that have Kikihia only have Kikihia. Again, make sure to run this command in the fullAHEloci folder.
```
while read x; 
 do mv $x ./rmtaxaout/$x;
 done < nonKfiles
```
Now that we have done this, we want to reorganize each file so that the longest sequence comes first, since we want to use the longest possible query sequences.
```
cd rmtaxaout
python /home/CAM/egordon/scripts/taxon_regroup.py -seqlen .
```
The reorganized files are in a new folder called regrouped. We now have a full list of queries to blast against our assemblies.



#### Step 3: BLAST
Before we blast our queries against our assemblies, we need to make sure that our assemblies have blast databases associated with them. You need to have a folder that contains all of your contigs.fasta files from each of your assemblies named based on the sample number. Look in that folder, and if each of the <sample>contigs.fasta files also has a <sample>contigs.fasta.nhr, .nin, and .nsq file, you have already formatted them as blast databases. This should be the same folder that we had for the blast databases when we ran through this the last time. If you need a reminder on how that step is done, it is something like this:
```
module load blast
for x in *.fasta; do makeblastdb -in $x -dbtype nucl; done
```
If your folder has already been formatted to contain blast databases, you can now submit a job that uses Eric's folder blast script. For reference, the folder blast script takes your folder of reorganized files (to be used as queries), your folder with your blast databases, and a file listing the assemblies it is going to use. To make that list, in your regrouped folder run the following:
```
ls <path to folder containing assemblies formatted as blast databases>/*.fasta > LIST
sed -i -r 's/.+I/I/g' LIST
```
The sed command is to remove the extra folder path information that ls prints into LIST that will mess up the folder blast script. You can use RegExr to customize it until it removes the folder path information before the actual name of the file.

Once you have that list, create or modify a script to look something like this and submit it. It will take a while to run.
```
#!/bin/bash
#SBATCH --job-name=blast
#SBATCH -N 1
#SBATCH -n 1
#SBATCH -c 12
#SBATCH --partition=general
#SBATCH --qos=general
#SBATCH --mem=50G
#SBATCH --mail-type=END
#SBATCH --mail-user=mark.stukel@uconn.edu
#SBATCH -o myscript_%j.out
#SBATCH -e myscript_%j.err
module load blast
cd <your filepath>/fullAHEloci/rmtaxaout/regrouped/
bash /home/CAM/egordon/scripts/folder_blast.sh ./ <your folder with your blast databses> 1e-10 tblastx 12 y LIST
```


#### Step 4: Parsing the BLAST results
Once the script finishes, you should have a number of .blast files in your regrouped folder. Create a blast results folder and move them there for the blast parser script later.
```
mkdir blastresults
for x in *.blast; do mv $x blastresults/$x; done
```

Now it's time to run the blast parser script. This script takes the folder with the blast results, the folder with all the assemblies (you can use the blast databases folder), the folder with the blast queries, an e-value cutoff, an option for how much sequence to extract, and how many blast hits to extract per query. We will be using the -me500 option, which extracts a 500 nucleotide buffer on each side of the blast hit for multiple .blast files, which seems to be a good buffer for aligning the extracted hits later. You can see the other options by running the script without any arguments.
```
module load python/2.7.8
python /home/CAM/egordon/scripts/alexblastparser.py blastresults/ <folder with contigs.fasta files> ./ 1 -me500 1
```
There should now be a new folder called modified containing the extracted hits. They are organized according to query sequence, with the top hit from all the samples. Lets use the alignment program mafft implemnted in another script that takes the arguments folder with fasta file, algorithm(ginsi einsi linsi), whether to adjust directions or not (adjust, noadjust, slow) , and number of threads. Aligned files are output to "realigned" folder. Inside the new modified folder, create and run a script that looks something like this:
```
#!/bin/bash
#SBATCH --job-name=align
#SBATCH -N 1
#SBATCH -n 1
#SBATCH -c 8
#SBATCH --partition=general
#SBATCH --qos=general
#SBATCH --mem=50G
#SBATCH --mail-type=END
#SBATCH --mail-user=mark.stukel@uconn.edu
#SBATCH -o myscript_%j.out
#SBATCH -e myscript_%j.err
bash /home/CAM/egordon/scripts/align.sh ./ einsi adjust 8 y
```
This script will take a while as well.


#### Step 5: Visualizing the alignments
Inside the realigned folder, you will see all the newly aligned genes. Instead of looking at each of the 500+ files individually, we can run an R script that outputs a nice visualization of the alignments so we can quickly see how well the taxa are aligned for each gene. Because of recent changes to how R is set up on the cluster, we need to install the necessary packages into our own folder before we run the script. We should have done this already, but here is what we did as a reference.
```
mkdir ~/R-libs #this is the directory that the packages we install will be stored
module load R
R #starts an interactive session of R
install.packages("ape", repos="http://cran.r-project.org", lib="~/R_libs/")
install.packages("phangorn", repos="http://cran.r-project.org", lib="~/R_libs/")
install.packages("seqinr", repos="http://cran.r-project.org", lib="~/R_libs/")
q() #quits interactive session
```
The final preparation step is to make sure the alignment script loads these packages from our ~/R_libs folder. I have already done this in my folder, but basically what you need to do is edit the script to change all instances of
```
library(<package>)
```
with
```
library(<package>, lib="~/R_libs")
```
To run the R script, we need to be on a compute node instead of the head node. To do this, run
```
srun -c 1 --partition=general --qos=general --mem=2G --pty bash
```
Technically you should be running interactive jobs on a compute node instead of the head node, but it looks like R requires you to be one one in order to actually run properly. You can return to the head node at any time by typing "exit".

An issue that the script sometimes runs into is empty files. Before you run the script, check to see if there are any loci that do not have any sequences in them. 
```
du *.fas
```
This will display the filesize in kB for each alignment. View any alignment that is <10 kB with cat to check if it is empty. If there are any empty files, remove them using rm before running the script.

You can now run the R script. You can change the number at the end to adjust how many alignments will be displayed per .png file, which changes how many files get created to view all the alignments.
```
Rscript /home/CAM/mstukel/scripts/view_DNAalignments.R -d . 30
```
You can now download and view the resulting .png files to assess how good your alignments are!

#### Step 6: obtaining summary statistics on alignments
Something that we will need to periodically do throughout our trimming process is get summary statistics about our alignment, usually in the form of average number of taxa per aligment and average alignment length. We will use some for loops and an awk command to do this.

To calculate the average number of taxa among our alignments, we first write a for loop counting up all the lines that had a blast result (have ".fasta" in the sequence header) and appending them to the end of a "stats" file.
```
for x in *.fas; do grep '.fasta' $x | wc -l >> stats; done
```
Now we run the awk command to find the mean for all the loci. This command adds together all the values in the first column and then divides the total by the number of rows.
```
awk '{ total += $1 } END { print total/NR }' stats > summarystats
```
This command is super useful in other contexts for averaging a column of data in different tables. To change what column it is averaging, change "$1" to whatever number your column of interest is.

Now we will get the alignment lengths. The best way for this is to find the length of the longest line in each alignment with the -L flag in wc.
```
for x in *.fas; do wc -L $x >> length; done
```
Now run the same awk command above and append it to the end of summary stats.
```
awk '{ total += $1 } END { print total/NR }' length >> summarystats
```
We now have a summary stats file with the average dimensions (in rows and columns) of all our alignments!

#### Step 7: Splitting paralogs
We have a problem with paralogs in our alignments. This is caused by issues with the BLAST step earlier. For some loci, the ortholog of the query sequence was not the top BLAST hit for some of the taxa. This could either be caused by the ortholog not being present in that taxon's assembly or by the query sequence having issues like ambiguity codes or being too distant. In either case, the the BLAST parser script took the top hit and included it in the alignment, even though it might not have been orthologous. 

The workflow for sorting out these paralogs is as follows: First, scan through the visualized alignments from Step 5 to find alignments were it appears that several sequences do not match the others. Load those alignments into the MAFFT web server and use the phylogenetic tree option to generate a quick NJ tree based on sequence similarity. Examine the tree to see whether the branching pattern makes sense, or if the taxa are divided into two or more distinct monophyletic groups separated by long branches. If the taxa are divided into clades with long branches, take a sequence from each clade and run a blastx search. If the blast results for the sequences from each clade appear to be different proteins, the alignment needs to be split. Divide the alignment into two or more alignments based on the number of paralogs found, with a ".1" after the locus number in the filename for the alignment with the most taxa, a ".2" for the next most, and so on. While we will use the ".2" and ".3" files later after more processing, we can use the ".1" files for now. Copy all the alignment files to a new folder. If an alignment has been split into multiple files, include only the ".1" file in this new folder, not the original or the other paralogs.

#### Step 8: removing query sequences from the files
Our problem is that we have a bunch of not-particularly-related query sequences in our files that we would like to remove so that it does not screw up our HmmCleaner analysis. We will be using the removeTaxa.py script that we used way back when to do it.

One quick thing we should do before we start is clean up the header names of sequences that were reversed. MAFFT puts a "\_R\_" in front of sequences that it reverses during the alignment process. We shouldn't have too many of those, but it should be easy to get rid of those characters using ```sed```.
```
for x in *.fas; do sed -i 's/_R_//' $x; done
```
Our next step will be to make a list of all the fasta file headers. We will then narrow this down to just the taxa we want to remove. It should be easy to do this using ```grep```. ```grep``` prints the line that contains the string you are searching for, so we can just search for the ">" that starts each header line and append the output to a list of all the headers.
```
for x in *.fas; do grep '>' $x; >> headerlist; done
```
One problem that we have is that our headerlist file is huge and full of duplicates from all the different files. We can use an ```awk``` command that I found to remove the duplicates and put the output to a new file.
```
awk '!seen[$0]++' headerlist > taxatoremove
```
You can look at the taxatoremove file to see that the duplicates have been removed. Now we want remove all the taxon names from this file that correspond to taxa from our assemblies (we want to keep these, so we are removing them from this list of taxa to remove). Since all of the taxon names that come from our assemblies have "contigs.fasta" in the name, a ```sed``` command should be sufficient.
```
sed -i '/contigs.fasta/d' taxatoremove
```
We now have our file listing our taxa to remove. We can now run the removeTaxa.py script with the "exclude" option.
```
python /home/FCAM/egordon/scripts/removeTaxa.py -e taxatoremove
```
Like before, this creates a new folder called "rmtaxaout" with the output files.

#### Step 9: Realign
We should probably realign the sequences after removing the query sequences, just to be sure before we do any serious trimming. Copy your realign script from your "modified" folder up a few folder directories into this folder and run it.

#### Step 10: HmmCleaner
You can now move the HmmCleaner script you used to the new realigned folder and run it.

After HmmCleaner finishes, it will create a bunch of new files with "hmm" in their name in the "realigned" folder you ran the hmmclean.sh script from. Make a new directory called something like "hmmcleaned" and use the ```mv``` command to move all of those files there. You should notice that there are ```.fasta```, ```.log```, and ```.score``` files there. You don't have to worry about the ```.log``` and ```.score``` files. You will need to change the ```.fasta``` files to ```.fas``` files using the ```basename``` command. You can look up the syntax for changing the file extension of a file using ```basename```.

#### Step 11: Visualizing Hmmcleaned files
Use the R visualization script just like we did earlier in this tutorial to visualize the cleaned files.

#### Step 12: Trimming missing data
We can use the ```customtrim.py``` scripts in my scripts folder to trim missing data off the ends. We will be using the "-%" option, which removes base positions from the ends of the alignments until it gets to a position that has data for a certain percentage of taxa. I was having trouble with this script before because the documentation was very vague, but after reading the script a few times I figured out what was going wrong. We actually specify the _percentage of taxa with missing data_ we want there to be, not _the percentage of taxa without missing data_. When I entered 90% before, it wasn't looking for positions that had 90% no missing data, it was looking for positions that had less than 90% missing data (10% of positions with missing data).

```
python /home/CAM/mstukel/scripts/customtrim.py . -%.1
```
The way the script works is it takes the first 2 characters of "-%.1" and sees that it is the option for percentage, and then it reads the characters afterwards as the proportion of missing data we want to have or less (0.1, or 10%). After the script runs, it creates a "trimmed" folder, where you can re-run the visualization script to inspect its handiwork.

#### Step 13: Creating species and gene trees
Now we can actually build trees. We will first build a concatenated tree of all loci that is partitioned by gene. Before we concatenate, we want to change the sequence names from "I-numbercontigs.fasta" to the actual taxon names. In your trimmed folder, create a file called rename.sh and put this command in it:
```
counter=0;for f in `cat lookup.txt`;do replace[counter]=$f;counter=$counter+1;done;counter2=0; for ((a=0; a <= $counter-1;a=a+2)); do `sed -i "s/${replace[$a]}/${replace[$a+1]}/g" *.fas` ;done;
```
Before running it, we need to have the lookup.txt file. In our Google Drive folder there is a file called lookup.txt. When you download it, it will save as an Excel file by default. After saving it, open it in Excel and save it just as a .txt file. Upload it to your trimmed folder. Now you can just run the script like this:
```
bash rename.sh
```
It might take a little while.

Now we can concatenate the files together. We can use the concat.py script in my scripts folder to do this.
```
python /home/CAM/mstukel/scripts/concat.py . -1
```
This option tells the script to not partition by codon, but only by locus. It creates an alignment of taxa in what is called the  "Phylip" format instead of the FASTA format that we have been used to. It also creates what is called a partition file. The partition file identifies what different parts of the alignment are. In this case, it identifies which base positions of the concatenated alignment correspond to which original loci. This is important for creating trees.

We are now ready to create trees. Make a script called concatenatetree.sh and put the following in it (changing the email to yours, of course):
```
#!/bin/bash
#SBATCH --job-name=iqtree
#SBATCH -N 1
#SBATCH -n 1
#SBATCH -c 16
#SBATCH --partition=general
#SBATCH --qos=general
#SBATCH --mem=50G
#SBATCH --mail-type=END
#SBATCH --mail-user=mark.stukel@uconn.edu
#SBATCH -o myscript_%j.out
#SBATCH -e myscript_%j.err
module load iqtree
iqtree -s COMBINED.phy -spp partitions.prt -bb 1000 -alrt 1000 -nt AUTO -m MFP+MERGE -rcluster 10 -pre maoricicada.merge
```
You can type "iqtree --help" before running the script to see what all of the options that I have provided mean. In short, what this script will do is identify which model of DNA evolution is best for each partition (since each locus might be different), and then tries to merge similar partitions together (since some loci might be evolving similarly). This will increase the accuracy of the tree that it creates. As a warning, this script will take a very long time to run (several days), which is why I specified 16 threads.

We also want to run individual gene trees as well. You need the beta version of IQTree for this, which I have installed in my own folder. Before we do so, create a folder inside your trimmed folder called "loci" and copy all of the .fas files into it. Create a second script called locitree.sh in your trimmed folder and put the following in it (changing the email again):
```
#!/bin/bash
#SBATCH --job-name=iqtree2
#SBATCH -N 1
#SBATCH -n 1
#SBATCH -c 8
#SBATCH --partition=general
#SBATCH --qos=general
#SBATCH --mem=50G
#SBATCH --mail-type=END
#SBATCH --mail-user=mark.stukel@uconn.edu
#SBATCH -o myscript_%j.out
#SBATCH -e myscript_%j.err
/home/CAM/mstukel/iqtree-2.0-rc1-Linux/bin/iqtree -S ./loci -nt AUTO -alrt 1000 -pre maoricicada.loci
```
What this script does is goes through all of the fasta files in the loci folder and creates separate gene trees for each one. The files have to all be in a separate folder because IQTree isn't smart enough to only look for fasta files when it looks through a folder. This script will also take a while to run.

#### Step 14: Collapsing Nodes
Before we run ASTRAL, we have to use Newick Utilities to collapse low-support nodes to improve ASTRAL's accuracy. Here is the link to the program website: http://cegg.unige.ch/newick_utils. There are older binaries of the program that work with older versions of MacOS (and you could always compile from source if you really really wanted to...), but I think the best course of action is to just install the linux version into your home directory on the cluster. 

Go to your home directory on the cluster with ```cd ~``` and use the ```curl``` command to copy the file like this:
```
curl -O http://cegg.unige.ch/pub/newick-utils-1.6-Linux-x86_64-disabled-extra.tar.gz
```
Since it is a tar.gz file, we will need to uncompress it. 
```
tar -xzvf newick-utils-1.6-Linux-x86_64-disabled-extra.tar.gz
```
There should now be a folder in your home directory called "newick-utils-1.6" with the different programs inside it all ready to run. Go back to the folder with your gene tree file. This file should have a list of trees with support values listed in SH-aLRT scores. We will collapse any node that has a score of under 33. 
```
~/newick-utils-1.6/src/nw_ed maoricicada.loci.treefile 'i & b < 33' o > maoricicada.loci_collapsed.treefile
```

#### Step 15: ASTRAL
Read throught the ASTRAL documentation here to familiarize yourself with the different options: https://github.com/smirarab/ASTRAL/blob/master/astral-tutorial.md

Load ASTRAL using the module load command
```
module load astral
```

On the cluster, the ASTRAL file location is stored in the ```$ASTRAL``` path variable, so you don't need to know exactly where it is stored. This means when you run ASTRAL, it will be in the form of ```java -jar $ASTRAL```.
Use the ```srun``` command to get on a compute node like you did in Step 5 before you run ASTRAL. A quirk about ASTRAL is that it outputs the tree to stdout and the log information to stderr. Make a new folder called ```astral``` and ```cd``` in there. This is where we will store all of our output.
To create a species tree from your gene trees, run
```
java -jar $ASTRAL -i ../maoricicada.loci_collapsed.treefile -o maoricicada.astral.tre 2> maoricicada.astral.log
```
(change the filenames/filepaths to match what you have)
The ```2>``` saves the stderr, which contains the log information. This should take only a couple of minutes. The ASTRAL documentation tells you what the log contains, but one of the more important numbers is the "normalized quartet score". This is the proportion of four-taxon quartets in the gene trees that are satisfied by the species tree. The closer this proportion is to 1, the less discordance there is among your gene trees. 
There is another option we can run that changes the branch labels. The default branch label is ASTRAL posterior probability, which gives a probability between 0 and 1 that the branch is a real branch. The documentation gives the different options for branch label information. You might be tempted to use ```-t 2``` which gives the full annotation, but I found in practice that the label is too hard to read. This time we will be using ```-t 8```, which gives the alternative quartet topologies.
```
java -jar $ASTRAL -i ../maoricicada.loci_collapsed.treefile -o maoricicada.astral_quartets.tre -t 8 2> maoricicada.astral_quartets.log
```
Notice that I changed the output and log filenames. Now the branch supports will be three numbers. At any branch in an unrooted tree, there are three possible ways to arrange 4 taxa around it (try it yourself on a piece of scrap paper with 4 taxa labeled A, B, C, and D). The first number shows the proportion of gene tree quartets that support the arrangement of taxa that is currently shown. In an ASTRAL tree, it should be a plurality of the quartets, or else ASTRAL would not choose that arrangement for the tree topology. The other two numbers are the proportions for the other two possible arrangements. This tells you whether most of the gene tree quartets agree with one topology, if there are two topologies that are well-supported, or if there is an almost even split.
We can also score an existing tree topology using ASTRAL. We use the ```-q``` option for this. Lets score the concatenated tree (change the filepaths/filenames to match what you have).
```
java -jar $ASTRAL -q ../maoricicada.merge.treefile -i ../maoricicada.loci_collapsed.treefile -o maoricicada.astral_concat.tre 2> maoricicada.astral_concat.log
```
```
java -jar $ASTRAL -q ../maoricicada.merge.treefile -i ../maoricicada.loci_collapsed.treefile -o maoricicada.astral_concat_quartets.tre -t 8 2> maoricicada.astral_concat_quartets.log
```
This will tell you how well your gene trees agree with your concatenated tree. Download all of the .tre files and view them in FigTree.

#### Step 16: RAxML gene trees
We used IQ-Tree to generate gene trees in our previous example. While IQ-Tree is really useful for generating trees with lots and lots of DNA sequence data or many, many taxa, it is not usually used to generate trees from individual genes. We can use RAxML, which is a program more commonly used for this purpose, to generate our gene trees. RAxML has a slower algorithm than IQ-Tree, and the models of evolution that it steers you towards picking are more complicated than what IQ-Tree suggests. 

In your ```trimmed``` folder, create a new folder called ```raxml``` and put the following in a new script in that folder.

```
#!/bin/bash
#SBATCH --job-name=raxml
#SBATCH -N 1
#SBATCH -n 1
#SBATCH -c 8
#SBATCH --partition=general
#SBATCH --qos=general
#SBATCH --mem=50G
#SBATCH --mail-type=END
#SBATCH --mail-user=[your email here]
#SBATCH -o myscript_%j.out
#SBATCH -e myscript_%j.err
module load RAxML
cd ../loci
for x in *.fas; do mkdir ../raxml/$x; cp $x ../raxml/$x; done
cd ../raxml
for x in *.fas; do cd $x; raxmlHPC -f a -x 12345 -p 12345 -N 200 -m GTRGAMMA -s $x -n TEST; cd ..; done
for x in *.fas; do cat $x/RAxML_bipartitions.TEST >> maoricicada.raxml.tre; done
```
This script creates a new folder in the raxml folder for each alignment file. It runs RAxML on each alignment file and saves all the output files to the corresponding folder. Finally, it collects all the individual tree files for each gene into a single file. You can now follow Steps 14 and 15 again, except using this file instead of the maoricicada.loci.treefile. Be sure to change the names of your output and log files so that they have raxml in the name so you don't get confused.

