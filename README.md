# Overview

These are the tools that I wrote for the UCSC Genocoding project, see
http://text.soe.ucsc.edu. They allow you to download fulltext research
articles from internet, convert them to text and run text mining algorithms
on them.  All tools start with the prefix "pub". 
This is a early testing release, please send error messages to Maximilian Haeussler, max@soe.ucsc.edu.

# The tools

- pubCrawl = crawl papers from various publishers, needs a directory with a
        textfile "pmids.txt" in it and the data/journalList directory
- pubGet<PUB> = download files from publisher PUB directly (medline, pmc, elsevier)
- pubConv<PUB> = convert downloaded files to my pub format (tab-separated table
             with fields defined in lib/pubStore.py)
- pubLoadMysql and pubLoadSqlite = load pub format data into a database system 
- pubRunAnnot = run an annotator from the scripts directory on text data in
             pub format
- pubRunMapReduce = run a map/reduce style job from "scripts" onto fulltext.
- pubLoad = load pub format files into mysql db
- pubMap = complex multi stage pipeline to find and map markers found in text 
           (sequences, snps, bands, genes, etc) to genomic locations 
           and create/load bed files into the ucsc browser
- pubPrepX = prepare directory structures. These are used to download
        taxon names, import gene models from websites like NCBI or
        UCSC. 

Most start with the prefix "pub", the category and then the 
data source or publisher. The categories are:

If you plan to use any of these, make sure to go over lib/pubConf.py first.
Most commands need some settings in the config file adapted to your particular
server / cluster system. E.g. pubCrawl needs your email address, pubConvX 
need the cluster system and various input/output directories.

# An example run

Create a directory

    mkdir myCrawl

Get a list of PMIDs, put them into the file pmids.txt

    echo 17695372 > myCrawl/pmids.txt

Run the crawler in unrestricted mode and with debug output on this list

    pubCrawl -du myCrawl/pmids.txt

The PDFs should be in the subdirectory myCrawl/files. Error messages are in myCrawl/pmidStatus.txt. 
Metadata is in a sqlite and a tab separated file. 

Convert crawled PDFs to text:

    mkdir myCrawlText
    pubConvCrawler myCrawl myCrawlText

# Output format

To allow easy processing on a cluster of metadata and text separately, the tools store the text as gzipped tab-sep tables, split into chunks with several hundred rows each (configurable). There two tables:
- articles.gz
- files.gz

The table "articles" contains basic information on articles. The internal article identifier integer numberthe internal article identifier integer number, an "external ID" (PII for Elsevier, PMID for crawled articles, Springer IDs for Springer articles, etc), the article authors, title, abstract, keywords, DOI, year, source of the article, fulltext URL, etc (see lib/pubStore.py for all fields). The internal article identifier (articleId) is a 10 digit number and is unique across all publishers and articles.

The table "files" contains the files, one or more per article: the ASCII content string, the URL for each file, the date when it was downloaded, a MIME type etc. All files also have a column with the external identifier of the article associated to it. The internal fileID is the article identifier plus some additional digits. To get the article for a file, you can either use the externalID (like PMID12345) or the first 10 digits of fileId. 

One article can have several fulltext files and several supplemental files. It should have at least one main file (even though in an old version of the tables, there were articles without any file, this should be corrected by now). 

This format allows it to use all the normal UNIX textutils. E.g. to search for all articles that contain the word HOXA2 and get their external IDs (which is the PMID for crawled data) you can use simply zgrep:
    zgrep HOXA2 *.files.gz | cut -f2 | less

As the files are sorted on the articleId, you can also create a big table that includes both meta information and files in one table by gunzipping all files first and then running a join:
    join 0_00000.articles 0_00000.files > textData.tab

# Installation

If text annotation is too slow for you:
The re2 library for python will make it at least 10 times faster.
You need to download the C++ source from re2.googlecode.com, compile and install it
"make;make install" (by default to /usr/local), then install the python wrapper
with "pip install re2". (If you don't have write access to /usr/local, you need
to install the re2 library with "make install prefix=<dir>" and then change
setup.py in the python re2 install package, replace "/usr" with <dir>)

# BUGS to fix:

fixme: illegal DOI landing page
http://www.nature.com/doifinder/10.1046/j.1523-1747.1998.00092.x

URL constructor:
http://www.nature.com/nature/journal/v437/n7062/full/4371102a.html
for DOI  doi:10.1038/4371102a

URL construction for supplemental files:
http://www.nature.com/bjc/journal/v103/n10/suppinfo/6605908s1.html

no access page:
http://www.nature.com/nrclinonc/journal/v7/n11/full/nrclinonc.2010.119.html
- in wget, it triggers a 401 error

cat /cluster/home/max/projects/pubs/crawlDir/rupress/articleMeta.tab | head
-n13658 | tail -n2 > problem.txt