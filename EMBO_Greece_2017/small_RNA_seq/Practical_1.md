# MicroRNA Target Analysis from Expression Data

* * *

In this practical we will be taking 8 samples of Solexa Sequencing data from the course samples that were prepared for small-RNA sequencing from total RNA.

It is important for next-gen sequence analysis to know the exact architecture of your sequencing reads including the 5' sequencing adapters and the sequences of barcoding tags and which samples each refers to

For our course data, the read architecture is as follows:

| 5' adpt | genomic RNA | 3' adpt |
| . | . |

Since miRNAs are shorter than the average length of a Solexa read (35-80nt), we will usually sequence through a microRNA sequence into the 3' adapter sequence. This needs to be detected and cleaned from our reads before they can be mapped. Additionally, each sequence will need to be sorted according to which barcode is present at the 5' end of the read into appropriate sample files.

Usually small RNA sequencing results in millions of reads for processing. The vast majority of tools for dealing with this kind of data require large amounts of memory and significant computing resources. However, on this course we will be testing a new ultra-fast read processor from the Enright lab, called **Reaper** which will be available in BioConductor shortly.

* * *

### Fastq file format

Most sequencing results can be obtained in a format called _fastq_, reflecting that they are _fasta_ files with _quality_ scores. These files contain the actual nucleotide bases for every sequence as well as a quality score which reflects how sure we are that each base is actually the correct one.

The quality score is usually given by a _Phred_ score, which is related logarithmically to the probability of making an error when base calling. The following table describes this:

| Phred score | Probability that the base is incorrect | Precision of the base |
| 10 | 1 in 10 | 90 % |
| 20 | 1 in 100 | 99 % |
| 30 | 1 in 1000 | 99.9 % |
| 40 | 1 in 10000 | 99.99 % |

Getting back to our fastq file, the first few lines look like this:

<pre class="res"> <tt>@GAII06_0003:3:1:1040:19544#0/1
CCAAAGTGCTGGAATCACAGGCGTGAGCTACCTCGCCTGGCCTGAATTATT
+
bb__Zbb^^^]Z_]]aT]`^TbbYb^_`T_cc`aac^Y\`b`\`L]_]\YX</tt> 
</pre>

The first line is the identifier (starting with @), the second the sequence. The third is an extra spacer line (which sometimes also has the identifier repeated). The fourth are _ascii_ encoded quality scores, where each _letter_ represents a different number.

This is the standard _ascii_ table that was used to encode these numbers:

<pre class="res"> <tt>0 nul    1 soh    2 stx    3 etx    4 eot    5 enq    6 ack    7 bel
       8 bs     9 ht    10 nl    11 vt    12 np    13 cr    14 so    15 si
      16 dle   17 dc1   18 dc2   19 dc3   20 dc4   21 nak   22 syn   23 etb
      24 can   25 em    26 sub   27 esc   28 fs    29 gs    30 rs    31 us
      32 sp    33  !    34  "    35  #    36  $    37  %    38  &    39  '
      40  (    41  )    42  *    43  +    44  ,    45  -    46  .    47  /
      48  0    49  1    50  2    51  3    52  4    53  5    54  6    55  7
      56  8    57  9    58  :    59  ;    60  <    61  =    62  >    63  ?
      64  @    65  A    66  B    67  C    68  D    69  E    70  F    71  G
      72  H    73  I    74  J    75  K    76  L    77  M    78  N    79  O
      80  P    81  Q    82  R    83  S    84  T    85  U    86  V    87  W
      88  X    89  Y    90  Z    91  [    92  \    93  ]    94  ^    95  _
      96  `    97  a    98  b    99  c   100  d   101  e   102  f   103  g
     104  h   105  i   106  j   107  k   108  l   109  m   110  n   111  o
     112  p   113  q   114  r   115  s   116  t   117  u   118  v   119  w
     120  x   121  y   122  z</tt> 

</pre>

From this table, we can see that the very frequent quality score "b" actually represents a numerical value of 98\. One small detail: since the first 0-32 ascii codes represent strange things (e.g. bell, new line, backspace) we cannot use them for encoding. Thus, in order to encode real quality scores (0-40) we first need to shift the quality scores to avoid these strange characters. Unfortunately, there are two current standards, one which shifts the quality scores by adding 33, another by adding 64\. The file we'll be using has been shifted by 64. This means that "b" actually represents the quality score of 34 (98 - 64).

<pre> <tt>Some _ascii_ characters are unprintable so the entire table is shifted by 33 giving a final lookup table as follows
where each symbol represents a unique Phred score.
 <tt>0      !        1      "        2      #        3      $        4      %        5      &        6      '
     7      (        8      )        9      *       10      +       11      ,       12      -       13      .       14      /
    15      0       16      1       17      2       18      3       19      4       20      5       21      6       22      7
    23      8       24      9       25      :       26      ;       27      <       28      =       29      >       30      ?
    31      @       32      A       33      B       34      C       35      D       36      E       37      F       38      G
    39      H       40      I       41      J       42      K       43      L       44      M       45      N       46      O
    47      P       48      Q       49      R       50      S       51      T       52      U       53      V       54      W
    55      X       56      Y       57      Z       58      [       59      \       60      ]       61      ^       62      _
    63      `       64      a       65      b       66      c       67      d       68      e       69      f       70      g
    71      h       72      i       73      j       74      k       75      l       76      m       77      n       78      o
    79      p       80      q       81      r       82      s       83      t       84      u       85      v       86      w
    87      x       88      y       89      z</tt></tt> </pre>

<tt>

* * *

The module is available [here](data/Reaper_1.5.tar.gz) If you want to install this in your own R at home.

The fastq files, pdata file and mircounts file are also [available](data/course_miseq.tar.gz) but preinstalled on these machines.

<pre class="com">library(Reaper)
library(gplots)
library(RColorBrewer)
</pre>

Now we will set our working directory to where the solexa FASTQ files (zipped) are stored

<pre class="com">
setwd("~/Desktop/course_miseq")
list.files()
</pre>

Hopefully, you will see a compressed FASTQ txt file for each of the 4 lanes

<pre class="res"> [1] "lane1Brain2_sequence.fastq.gz"    "lane1Brain5_sequence.fastq.gz"   
 [3] "lane1Brain6_sequence.fastq.gz"    "lane1Heart1_sequence.fastq.gz"   
 [5] "lane1Heart11_sequence.fastq.gz"   "lane1heart3_sequence.fastq.gz"   
 [7] "lane1Heart8_sequence.fastq.gz"    "lane1Liver10_sequence.fastq.gz"  
 [9] "lane1Liver12x2_sequence.fastq.gz" "lane1Liver4_sequence.fastq.gz"   
[11] "lane1Liver7_sequence.fastq.gz"    "lane1Liver9_sequence.fastq.gz"   
[13] "mircounts.txt"                    "pdata.txt"   
</pre>

It is important that we also load information for reaper that tells it the following:

*   Which FASTQ files are present.
*   Which Barcode sequences correspond to which sample names.
*   What 5' and 3' sequencing adapters were used in library generation.

<pre class="com">pdata <- read.table("pdata.txt",header=TRUE,check.names=FALSE)
pdata
</pre>

You should see the following table loaded into R:

<tt>

<pre class="res">	"filename"		    "3p-ad"                      "tabu" "samplename"
1     lane1Brain2_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Brain
2     lane1Brain5_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Brain
3     lane1Brain6_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Brain
4    lane1Heart11_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Heart
5     lane1Heart1_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Heart
6     lane1heart3_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Heart
7     lane1Heart8_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Heart
8    lane1Liver10_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Heart
9  lane1Liver12x2_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Liver
10    lane1Liver4_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Liver
11    lane1Liver7_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Liver
12    lane1Liver9_sequence.fastq.gz AGATCGGAAGAGCACACGTCTGAACTCC   NA	Liver
</pre>

</tt>

Next we will start the Reaper algorithm. It will perform the following functions on all the lanes we have provided:

*   Splitting up reads according to provided barcodes
*   Detection of 3' or 5' adapter contamination using Smith-Waterman local alignment
*   Detection of low-complexity sequences, such as Poly-As or Poly-Ns
*   Quality score thresholding and trimming if required
*   Collapsing of reads according to depth in a summary FASTA result file per barcode per lane.
*   Generation of Quality Control plots for assessing sequencing quality

Reaper is started by passing it our samples table and telling it which "mode" to run in, in this case the mode is set to: **no-bc**.

<pre class="com"># For the sake of time we'll only run on the first 100000 reads
reaper(pdata,"no-bc",c("do"="100000"));

# This is the command to do all reads
# reaper(pdata,"no-bc"); We comment it out here
</pre>

<tt>

<pre class="res" style="background-color:#000000;color:#FFFFFF;">[1] "Starting Reaper for file: lane1Brain2_sequence.fastq.gz"
                          fastq                            geom 
"lane1Brain2_sequence.fastq.gz"                         "no-bc" 
                           meta                        basename 
                "metadata1.txt"                             "1" 
Passing to reaper: dummy-internal --R -fastq lane1Brain2_sequence.fastq.gz -geom no-bc -meta metadata1.txt -basename 1

---
mRpm   million reads per minute
mNpm   million nucleotides per minute
mCps   million alignment cells per second
lint   total removed reads (per 10K), sum of columns to the left
25K reads per dot, 1M reads per line  seconds  mr mRpm mNpm mCps {error qc  low  len  NNN tabu nobc cflr  cfl lint   OK} per 10K
........................................   26   1  2.4  115   59    0    0    0    0    0    0    0    0    0    0 10000

</pre>

</tt>

Cleaning many millions of reads will take some time, the method processes around 2M-4M reads per minute.

Reaper is designed to be fast and memory-efficient so it should run on any machine with 500MB of RAM or more. The time taken to complete the run depends on how fast the processors in your machine are.

Let's take a look at the Quality Control Metrics generated

<pre class="com">X11()
par(ask=TRUE)
reaperQC(pdata)
</pre>

<pre class="com"># Lets also make a nice PDF of the results
pdf("reaper.pdf",width=12)
reaperQC(pdata)
dev.off()
</pre>

You should get one plot back for each lane processed.

![](images/reaper_image1.jpg)

Here is a list of files generated during the Reaper run

<pre class="res">10.lane.clean.gz
10.lane.clean.uniq.gz
10.lane.report.clean.len
10.lane.report.clean.nt
10.lane.report.input.nt
10.lane.report.input.q
10.lint.gz
10.sumstat
11.lane.clean.gz
11.lane.clean.uniq.gz
11.lane.report.clean.len
11.lane.report.clean.nt
11.lane.report.input.nt
11.lane.report.input.q
11.lint.gz
11.sumstat
12.lane.clean.gz
12.lane.clean.uniq.gz
12.lane.report.clean.len
12.lane.report.clean.nt
12.lane.report.input.nt
12.lane.report.input.q
12.lint.gz
12.sumstat
1.lane.clean.gz
1.lane.clean.uniq.gz
1.lane.report.clean.len
1.lane.report.clean.nt
1.lane.report.input.nt
1.lane.report.input.q
1.lint.gz
1.sumstat
2.lane.clean.gz
2.lane.clean.uniq.gz
2.lane.report.clean.len
2.lane.report.clean.nt
2.lane.report.input.nt
2.lane.report.input.q
2.lint.gz
2.sumstat
3.lane.clean.gz
3.lane.clean.uniq.gz
3.lane.report.clean.len
3.lane.report.clean.nt
3.lane.report.input.nt
3.lane.report.input.q
3.lint.gz
3.sumstat
4.lane.clean.gz
4.lane.clean.uniq.gz
4.lane.report.clean.len
4.lane.report.clean.nt
4.lane.report.input.nt
4.lane.report.input.q
4.lint.gz
4.sumstat
5.lane.clean.gz
5.lane.clean.uniq.gz
5.lane.report.clean.len
5.lane.report.clean.nt
5.lane.report.input.nt
5.lane.report.input.q
5.lint.gz
5.sumstat
6.lane.clean.gz
6.lane.clean.uniq.gz
6.lane.report.clean.len
6.lane.report.clean.nt
6.lane.report.input.nt
6.lane.report.input.q
6.lint.gz
6.sumstat
7.lane.clean.gz
7.lane.clean.uniq.gz
7.lane.report.clean.len
7.lane.report.clean.nt
7.lane.report.input.nt
7.lane.report.input.q
7.lint.gz
7.sumstat
8.lane.clean.gz
8.lane.clean.uniq.gz
8.lane.report.clean.len
8.lane.report.clean.nt
8.lane.report.input.nt
8.lane.report.input.q
8.lint.gz
8.sumstat
9.lane.clean.gz
9.lane.clean.uniq.gz
9.lane.report.clean.len
9.lane.report.clean.nt
9.lane.report.input.nt
9.lane.report.input.q
9.lint.gz
9.sumstat
</pre>

#### Mapping Cleaned Reads to MicroRNAs

We will now use a web-utility to map cleaned and filtered reads against miRBase known mature miRNA sequences. This server will take each cleaned unique read and compare it to precursor sequences downloaded from miRBase. In the case of mismatches, a read will still be assigned if it has less than two mismatches assuming both sequences are each others best hit. The system supports compressed (GZ or ZIP) files as well as FASTQ text files.

Click [here](http://www.ebi.ac.uk/research/enright/software/chimira) To use Chimira

For this web-server, choose each of the **.clean.uniq** fastq files that were produced by reaper. Make sure you choose **Human** as the reference species

* * *

We now have raw counts of reads on microRNAs ready for QC and differential analysis.

</tt>