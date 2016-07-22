### Drug Interaction Discovery with NLP and Machine Learning:

---

__[Alexander L. Hayes](http://batflyer.net) | Savannah Smith | Devendra Dhami | Sriraam Natarajan__

Using data pulled from rxlist.com, openFDA, and PubMed.  This repository contains the scripts for pulling data from each source, and an overview of each are explained below.

Questions?  Contact Alexander Hayes at __hayesall(at)indiana(dot)edu__.

---

##### Table of Contents
1. [openFDA](#openfda)
2. [PubMed](#pubmed)
3. [Confidence](#confidence)
4. [Learning](#learning)

---

#####openFDA:
Here you will find several shell scripts for pulling drug names from rxlist.com, pulling labeling information from openFDA, and `fixlist.sh` to fix the list of drugs if `rxdownloader.sh` crashes halfway through.

Running the scripts will take some time (`rxdownloader.sh` took about 6 hours in total to crawl through the database), find out more about openFDA at [their website.](https://open.fda.gov/api/reference/)

The full set of [extracted data](https://github.iu.edu/hayesall/PMDataDump/tree/master/bashscripts/drugInteractionsFolder) can be found on Alexander's GitHub.  Because of its size, downloading a .zip is recommended.

1. `builddruglist.sh` [View Code](https://github.iu.edu/ProHealth/2016-ProHealthREU-DrugInteractionsDiscovery-SmithHayes/blob/master/openFDA/builddruglist.sh)

   Downloads drug names from rxlist.com  
   Outputs a text file (`drugslist.txt`)  
   Arguments can be passed to tweak the output.

  * `bash builddruglist.sh         `(by default formats for openFDA)
  * `bash builddruglist.sh openFDA `(replace spaces with +AND+)
  * `bash builddruglist.sh PubMed  `(replace spaces with +)
  * `bash builddruglist.sh Web     `(replace spaces with _)

2. `rxdownloader.sh` [View Code](https://github.iu.edu/ProHealth/2016-ProHealthREU-DrugInteractionsDiscovery-SmithHayes/blob/master/openFDA/rxdownloader.sh)

   Takes a list of drugs(drugslist.txt)  
   Queries openFDA for each drug (`fdainteractions.sh` [View Code](https://github.iu.edu/ProHealth/2016-ProHealthREU-DrugInteractionsDiscovery-SmithHayes/blob/master/openFDA/fdainteractions.sh))  
   Outputs a text file in drugInteractionsFolder/ (i.e. Warfarin+AND+Sodium will be output as drugInteractionsFolder/Warfarin+AND+Sodium-data.txt).

   Running it is simple (though time consuming):  
   `bash rxdownloader.sh`

   Additionally, RXDownloader does some sorting for us: it queries all __generic__ drugs, outputs __brand name__ drugs to a separate file (drugInteractionsFolder/BRANDNAMEDRUGS.txt), and separates __unknown drugs__ as well (drugInteractionsFolder/UNKNOWNDRUGS.txt).  Looking up generic drugs also pulls their brand-name equivalents, so redundancy is minimized.  Finally, each step is detailed in a Log file (drugInteractionsFolder/LOG.txt) which outlines what was queried and when it was completed.

   If RXDownloader crashes, it can be started up again to continue where it left off.

3. `fixlist.sh` [View Code](https://github.iu.edu/ProHealth/2016-ProHealthREU-DrugInteractionsDiscovery-SmithHayes/blob/master/openFDA/fixlist.sh)

   For error checking: run `bash fixlist.sh` to download a fresh copy of drugslist.txt, check each entry against the LOG file generated by RXDownloader, and remove any entries that are present in both.

[Return to Top](#drug-interaction-discovery-with-nlp-and-machine-learning) | [View in Folder](https://github.iu.edu/ProHealth/2016-ProHealthREU-DrugInteractionsDiscovery-SmithHayes/tree/master/openFDA)

---

#####PubMed:
The second dataset consists of Medical Abstracts from [PubMed](http://www.ncbi.nlm.nih.gov/pubmed).  The thought is that if drug-drug pairs appear in medical abstracts, the combination has been studied.  From the abstracts' text, we can discern the findings and whether or not there are adverse events caused by taking both.

Presented here are several scripts for pulling the abstracts.  The bulk of the work is done through the RefSense package (Maintained by Lars Arvestad and distributed under a GNU Public License: [[Website](http://www.csc.kth.se/~arve/code/refsense/) | [GitHub](https://github.com/arvestad/refsense)]).  

Professor Natarajan suggested we extract the top twenty abstracts from the past ten years (numbers chosen based on a paper he worked on), querying every drug combination (a total of 11,912,080 possible) and outputing the results to text files.  Once again, the full set of [extracted data](https://github.iu.edu/hayesall/PMDataDump/tree/master/Generated/Abstracts) can be found on Alexander's GitHub (~25 GB).

1. `pmid2text`, `pmsearch`, and perlscripts/

  * Each article in PubMed is associated with a unique ID number.  
  * `pmid2text` can be invoked to search for specific keywords and return all PubMed IDs associated with them.  Additionally, the `t` flag can be invoked to specify how old the abstracts can be (in this case, `-t 3650` for approximately 10 years), and the `d` flag can be passed to specify how many articles are returned (`-d 20`).  
  * `pmid2text` is used to convert these unique PubMed IDs to a text output: `-a -i` can be passed to pull the abstracts, and remove indentation, respectfully.

  * We can chain these together:  
   `perl pmsearch -t 3650 -d 20 $DRUG1 $DRUG2 | perl pmid2text -a -i > outputFile`

  * For completeness, additional functions are in the perlscripts/ directory.  Normally these would be placed under lib/, the path variable is updated automatically by the scripts.

2. `smartsplit.sh` [View Code](https://github.iu.edu/ProHealth/2016-ProHealthREU-DrugInteractionsDiscovery-SmithHayes/blob/master/PubMed/smartsplit.sh)

   The drug combinations can be thought of as a matrix with drugs on the x and y axes.  
   With n=4881 drugs, checking every combination would normally take n^2 time.  However, a simple property allows us to cut this number in half, because [(x & y) == (y & x) when x=/=y].
   
   However, running 11,912,080 checks in sequence would take close to 68 days (at least if you're running it on my laptop).  
   We needed to run lots of the checks in parallel, and we needed to split the matrix in a way that made sure each node was responsible for a roughly equal number of calculations.  
   
   ![Graph displaying how a matrix has duplicates when cut in half](http://i.imgur.com/cscwYOO.png "Cutting a matrix in half")
   This graph represents what we are interested in.  The shaded red section represents duplicates and the unecessary checks where x=y.  The shaded green columns show roughly equal areas: columns [A-H] have roughly the same number of checks as column [Z].  
   `smartsplit.sh` needs to be size aware, splitting the matrix into equal parts depending on how many nodes are available to calculate.

   `bash smartsplit.sh STABLE.txt` creates 71 directories under a directory called `Data/`.  (71 is the max number of nodes that can be allocated at once on IU's Odin Supercluster, more on that later).  Each directory under `Data/` is named after a number between 1 and 71.  Each of these subdirectories contains a copy of `STABLE.txt`, `drugs.txt`, and a `check_[1-71]`.

   A brief overview of each file (these are important for `pullabstractsODIN.sh`).  
  * `STABLE.txt`: a copy of druglist.txt, the output of `bash builddruglist.sh PubMed`.  It is called STABLE because it is never altered by a program, it is used for copying but is never modified.  
  * `drugs.txt`: a copy of `STABLE.txt` that can be altered by other scripts (file is the y axis).
  * `check_[1-71]`: this file is a list of drugs that a particular node is responsible for (file is the x axis).  Each item in this list is checked against everything in `drugs.txt` until check_[1-71] is exhausted.
 
3. `smartsplit`

[Return to Top](#drug-interaction-discovery-with-nlp-and-machine-learning) | [View in Folder](https://github.iu.edu/ProHealth/Drug_Interaction_Discovery/tree/master/openFDA)

---

#####Confidence:

[Return to Top](#drug-interaction-discovery-with-nlp-and-machine-learning) | [View in Folder](https://github.iu.edu/ProHealth/Drug_Interaction_Discovery/tree/master/openFDA)

---

#####Learning:

[Return to Top](#drug-interaction-discovery-with-nlp-and-machine-learning) | [View in Folder](https://github.iu.edu/ProHealth/Drug_Interaction_Discovery/tree/master/openFDA)

---
