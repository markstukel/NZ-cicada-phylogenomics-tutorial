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



