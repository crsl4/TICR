Goal: run the MrBayes and BUCKy steps (done by `mb.pl` and `bucky.pl`)
on a cluster that uses the job scheduler SLURM.

## mb.pl

Instead of the `mb.pl` script that parallelizes the MrBayes runs per gene, we want to use SLURM to parallelize the work. We have all the nexus files in the same folder, along with the following scripts:
- The julia script [`paste-mb-block.jl`](scripts-cluster/paste-mb-block.jl) will read all the
  nexus files in the directory, and will read a text file with the MrBayes block to paste
  onto all nexus files (option `‑m, ‑‑mb‑block` in the original `mb.pl` script).
  All the nexus files will be renamed: `1.nex, 2.nex, ...`
- The submit script [`mb-slurm-submit.sh`](scripts-cluster/mb-slurm-submit.sh) will parallelize
  all the individual-gene MrBayes runs with SLURM. Change `--array` to the correct number of genes.


In the original pipeline, `bucky.pl` runs `mbsum` on each gene,
and then `bucky` (using all genes) for each quartets.
Here we have two separate steps, explained below. 

## mbsum.pl

There is no such script yet, because this step is fast and the shell can suffice.
Suppose that we have a directory where each file is of the form `*.t` and is a MrBayes output file.
We can use `mbsum` to summarize a single `.t` file that correspond to one gene,
removing the first 1000 trees of each for burnin.

```shell
for X in *.t; do mbsum -n 1000 $X; done
```
This would create a file named <filename>.in for each file named <filename>.t .
Warning! It would overwrite files with the name <filename>.in if they exist.

Alternatively, the mbsum step can also be done with
the julia script [`mbsum-t-files.jl`](scripts-cluster/mbsum-t-files.jl).
Warning: this script is hard-coded to 3 independent runs per gene in MrBayes, and
for 2500 generations of burnin (both can be easily changed).

## bucky-slurm.pl 

The perl script [`bucky-slurm.pl`](scripts-cluster/bucky-slurm.pl) will run `bucky`
**just once**, for a single 4-taxon set (or quartet) and it will take as input:
  - mbsum folder name
  - output name
  - bucky arguments: alpha, ...
  - integer for the given quartet

It will produce, as output, a file for the given 4-taxon set with the
`.concordance` file and a file with the parsed output in the form of the CF table
(to append later).

The perl script [`bucky-slurm.pl`](scripts-cluster/bucky-slurm.pl) can be run by SLURM
with the submit script [`bucky-slurm-submit.sh`](scripts-cluster/bucky-slurm-submit.sh).
Change `--array` to the appropriate number of 4-taxon sets for your data.

**Note** the `bucky` executable needs to be placed in `/workspace/software/bin`.
Adjust the path as appropriate for your cluster.



