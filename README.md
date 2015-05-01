# GWCA
These three scripts can be utilized in series to perform highly parallized concordance analyses on any given alignment, with a particular focus on very large datasets which may include dozens of taxa and may span entire chromosomes or genomes.

## Dependencies
1. [PAUP*](http://paup.csit.fsu.edu/downl.html)
	* Used to perform quick parsimony analyses on all possible breakpoints to detect shifts in tree topology using MDL.
2. [mdl](http://www.stat.wisc.edu/~ane/programs/mdl1.0.tgz)
	* Places breakpoints in a given sequence in such a way that each partition has homologous tree topology.
3. [MrBayes](http://mrbayes.sourceforge.net/download.php)
	* Used to generate posterior distributions for each partition generated by MDL.
4. [mbsum and BUCKy](http://www.stat.wisc.edu/~ane/bucky/downloads.html)
	* mbsum: Summarizes the output from MrBayes into the required format for BUCKy.
	* BUCKy: Quantifies observed discordance between genes.

## Important Notes
* Even if you only plan on running these scripts only on a single computer, they still use ssh to spawn worker clients on the host computer. To ensure that there is no issue with this, enter the following command in your terminal:

    ```
ssh localhost
    ```

	If a prompt appears asking for host authenticity verification, answer with 'yes' and then you should not experience any issues.
* In order to function properly, any user-specified MrBayes block MUST allow logging to terminal.
* For "accurate" MCMC convergence checks, the user must ensure that their specified diagnose frequency divides evenly into their specified total number of MCMC generations.

## Supported Operating Systems
This script was tested and built on a cluster running Red Hat Enterprise Linux Server release 6.6 (Santiago), it should work for most other Linux distros, and might work on MacOS.

## Script Run Order
1. mdl.pl
	* This script handles the parallelization and execution of MDL. Upon completion, this script will have generated a gzipped tarball containing the sequence data of each partition found by MDL. This tarball is used as the input for the next script.
2. mb.pl
	* This script handles the parallelization and execution of MrBayes on all partitions located in the tarball generated by mdl.pl. Upon completion, this script will have generated a tarball containing the output from MrBayes for all partitions located in the input tarball. This tarball is used as the input for the next script.
3. bucky.pl
	* This script handles the parallelization and execution of BUCKy using the output from MrBayes stored in the tarball generated by mb.pl. Upon completion, this script will have generated a tarball containing the output from BUCKy for each possible quartet in the input sequence file. A summary of all concordance factors for each quartet will also be output in a csv file for user convenience.

## mdl.pl
### Script Work Flow
This script begins by reducing the given input alignment to only parsimony-informative characters using the program PAUP*. Once all parsimony-informative sites have been determined, PAUP* is used to calculate the parsimony score of every potential breakpoint which could occur in the input alignment. Using these parsimony scores, a Minimum Description Length criterion as implemented in the program mdl is utilized to determine where changes in tree topology across the alignment have occured (for a more in-depth overview of the principles behind MDL, read the following [paper](http://gbe.oxfordjournals.org/content/3/246.abstract?keytype=ref&ijkey=EGdYfRvyX3uU20z)). This effectively splits the input alignment into partitions of sequence with homologous topology. The start and end indices of these sequences are then converted back into their corresponding indices in the full alignment (non-parsimony informative characters included) which the user originally input. Non-parsimony informative characters located between two partitions are divided equally between them.
### Usage
At the minimum, this script requires a single FASTA or NEXUS file containing the alignment of interest. Additionally, the length of the smallest possible partition which MDL can find must also be specified with -b or --block-size option.  The script can then be called with the following invocation:

```
mdl.pl input.fa -b 100
```

This will then proceed to breakdown the alignment contained in input.fa into partitions having homologous topology and length of at least 100 parsimony-informative characters (potential  output alignments would contain could potentially contain any multiple of 100 parsimony-informative characters up to the total number of parsimony-informative characters present in the alignment).

For very large alignments (technically alignments with many parsimony-informative characters), such as those that would be expected from chromosomes or genomes, the number of parsimony informative calculations required to run MDL to completion using the above method becomes unfeasibly large. Therefore, it is recommended to use the -f or --forced-break option like so:

```
mdl.pl input.fa -b 100 -f 10000
```

Including this option as seen above would force a breakpoint after every 10,000th parsimony-informative character in the input alignment. Each block of 10,000 parsimony-informative characters would then have MDL run on it independently. The final partitioning is then a concatenation of the independent MDL analyses on the forced breakpoints.

### Command Line Options
The following table describes the support command-line optionswhich can be specified to influence the script execution:

| Option Flag(s) | Option Descripton | Default |
|:-|:-:|:-:|
| -b, --block-size | sets the minimum number of parsimony-informative characters which can be found in a block | none |
| -f, --forced-break | forces a breakpoint after the specified number of parsimony-informative characters, these partitions are then run independenly | none |
| --gap-as-char | specify to treat gaps as informative characters in PAUP* parsimony analyses| gaps not informative |
| --machine-file | file name containing hosts to ssh onto and perform analyses on, passwordless login MUST be enabled | none |
| --port | specifies the port to utilize on the server | 10001 |
| -o, --out-dir | name of the directory to store output files in | "mdl-" + Unix time of script invocation) |
| -T, --n-threads | the number of forks ALL hosts running analyses can use concurrently | current number of free CPUs |
| -h, --help |display help and exit | N/A |

### Output Files
Given an input alignment named 'chromosome1.fa', the following files can be found in the output directory upon successful completion of the script:

* **chromosome1-reduced.tar.gz**: zipped tarball containing the parsimony-informative character alignments for each forced breakpoint
* **chromosome1-scores.tar.gz**: zipped tarball containing the parsimony scores calculated for each possible breakpoint in each forced partition (or the whole alignment if no forced partitioning was used)
* **chromosome1-partitions.tar.gz**: zipped tarball containing the MDL output for each foreced partition of the alignment (or the whole alignment if no forced partitioning was used)
* **chromosome1.tar.gz**: contains the actual alignments which resulted from the MDL partitioning
* **chromosome1-stats.csv**: contains MDL partition start and end indices in terms of both the full and parsimony-informative character alignment, can be useful for determining average partition length, as well as the average number of parsimony-informative and non-parsimony informative characters present in each partition
* symlink to specified input file

## mb.pl
### Script Work Flow
This script begins by appending the user-specified MrBayes block to each Nexus file representing an MDL partition located in the input tarball created by mdl.pl. Once this is complete, the script proceeds to run MrBayes on each MDL partition. The results for each partition are then tarballed and gzipped. The tarballs for each partition are then aggregated into a single tarball.

### Usage
This script can be used to perform the parallelized execution of many MrBayes runs. Additionally, it can also be used to perform rudimentary MCMC convergence checks and remove non-convergent partitions.

#### Running MrBayes
When running MrBayes, this script utilizes the zipped tarball created by mdl.pl as its input. For example, if the you used the following command to partition an alignment:

```
mdl.pl chromosome1.fa -b 100 -f 10000
```

the required input for mb.pl would be named 'chromosome1.tar.gz'. Additionally, this script requires a file containing a properly formated MrBayes block (specified with -m or --mb-block flag). This MrBayes block contains the actual commands that will dictate how the script calls MrBayes. The following represents an example invocation:

```
mb.pl chromosome1.tar.gz -m bayes.txt -o chr1-mb
```

If the above command was for some reason cancelled, interrupted, or succesfully completed but perhaps had its output tarball modified using the --remove option, it is also possible to rerun only the needed partitions by specifying the previous output directory of the script as input like so:

```
mb.pl chr1-mb -m bayes.txt 
```

#### MCMC Convergence
The terminal output of each MrBayes run is by default stored by this script, if the original MCMC analysis were run with the following command:

```
mb.pl chromosome1.tar.gz -m bayes.txt -o chr1-mb
```

this information, as well as the resulting tree and parameter files for each MrBayes analysis, would be expected to be located a file named 'chromosome1.mb.tar'.

This output tarball can be parsed to determine the final standard deviation of split frequencies and roughly estimate whether or not the MCMC analyses which they describe reached convergence. If the user simply wants to check which partitions meet a specific threshold, they can use the -c or --check option like so:

```
mb.pl chr1-mb -c 0.05
```

The script will then proceed to output the final recorded standard deviation of split frequencies found in the log file of the MrBayes analysis for the partition and whether or not it met the specified threshold.

If the user wishes to remove partitions below a specific threshold (perhaps as a precursor to rerunning MrBayes with altered settings), they can use the -r or --remove option:

```
mb.pl chr1-mb -r 0.05
```

### Command Line Options
The following table describes the support command-line optionswhich can be specified to influence the script execution:

| Option Flag(s) | Option Descripton | Default |
|:-|:-:|:-:|
| -m, --mb-block | text file containing MrBayes commands to append to each input partition | none |
| -c, --check | outputs how many MrBayes runs standard deviation of split frequencies have reached the specified threshold | N/A |
| -r, --remove | removes MrBayes runs with standard deviation of split frequencies below the specified threshold | N/A |
| --machine-file | file name containing hosts to ssh onto and perform analyses on, passwordless login MUST be enabled | none |
| --port | specifies the port to utilize on the server | 10002 |
| -o, --out-dir | name of the directory to store output files in | "mb-" + Unix time of script invocation) |
| -T, --n-threads | the number of forks ALL hosts running analyses can use concurrently | current number of free CPUs |
| -h, --help |display help and exit | N/A |

### Output Files
Given an input partition tarball named 'chromosome1.tar.gz', the following files can be found in the output directory upon successful completion of the script:

* **chromosome1.mb.tar**: contains all output from the MrBayes analysis of each partition contained in 'chromosome1.tar.gz'
* symlink to specified input file

## bucky.pl
### Script Work Flow
This script begins by summarizing the MrBayes output files for each individual partition using the program mbsum. Once all posteriers for all partitions have been summarized, this script calculates all the possible quartets for the given alignment and runs each one through BUCKy do determine the concordance factors for the three possible splits.

### Usage
At the minimum this script only requires the tarball created by mb.pl as its input. For example, if the you used the following command to run MrBayes:

```
mb.pl chromosome1.tar.gz -m bayes.txt -o chr1-mb
```

the required input for bucky.pl would be named 'chromosome1.mb.tar'. The following represents an example invocation:

```
bucky.pl chromosome1.mb.tar -o chr1-bucky
```

If the above command was for some reason cancelled or interrupted, it is also possible to rerun only the needed quartets by specifying the previous output directory of the script as input like so:

```
bucky.pl chr1-bucky 
```

### Command Line Options
The following table describes the support command-line optionswhich can be specified to influence the script execution:

  -a, --alpha            value of alpha to use when running BUCKy (default: 1)      
  -n, --ngen             number of generations to run BUCKy MCMC chain (default: 1000000 generations)
  -o, --out-dir          name of the directory to store output files in (default: "bucky-" + Unix time of script invocation)
  -T, --n-threads        the number of forks ALL hosts running analyses can use concurrently (default: current number of free CPUs)
  --machine-file         file name containing hosts to ssh onto and perform analyses on, passwordless login MUST be enabled
                         for each host specified in this file
  --port                 specifies the port to utilize on the server (Default: 10003)
  -h, --help             display this help and exit
  --usage                display proper script invocation format

| Option Flag(s) | Option Descripton | Default |
|:-|:-:|:-:|
| -a, --alpha | value of alpha to use when running BUCKy | 1 |
| -n, --ngen | number of generations to run BUCKy MCMC chain | 1000000 generations |
| -r, --remove | removes MrBayes runs with standard deviation of split frequencies below the specified threshold | N/A |
| --machine-file | file name containing hosts to ssh onto and perform analyses on, passwordless login MUST be enabled | none |
| --port | specifies the port to utilize on the server | 10002 |
| -o, --out-dir | name of the directory to store output files in | "mb-" + Unix time of script invocation) |
| -T, --n-threads | the number of forks ALL hosts running analyses can use concurrently | current number of free CPUs |
| -h, --help |display help and exit | N/A |

### Output Files
Given an input tarball containing MrBayes output named 'chromosome1.mb.tar', the following files can be found in the output directory upon successful completion of the script:

* **chromosome1.mbsum.tar.gz**: contains the run output files produced during the summary of each MrBayes run with the program mbsum
* **chromosome1.BUCKy.tar**: contains the output files generated when running BUCKy on each quartet
* **chromosome1.CFs.csv**: contains the concordance factors for each possible split of each possible quartet as well as their 95% confidence intervals
* symlink to specified input file
