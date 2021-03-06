Pedigree Simulator
==================
Program to simulate pedigree structures. The method can use sex-specific
genetic maps and randomly assigns the sex of each parent (or uses user-specified
sexes) when using such maps.

Table of Contents
-----------------
   * Pedigree Simulator
      * [Basic usage](#basic-usage)
      * [Compiling](#compiling)
      * [Def file](#def-file)
      * [Map file](#map-file)
      * [Input VCF file](#input-vcf-file)
      * [Crossover model](#crossover-model)
      * [Output VCF file](#output-vcf-file)
      * [Output BP file](#output-bp-file)
      * [Output fam file](#output-fam-file)
      * [Output log file](#output-log-file)
      * [Sample ids for simulated individuals](#samp-ids)
      * [Extra notes: sex-specific maps](#extra-notes-sex-specific-maps)
      * [Citing Ped-sim](#citing-ped-sim)
      * [Other optional arguments](#other-optional-arguments)
         * [Specifying random seed](#specifying-random-seed---seed-)
         * [Genotyping error rate](#genotyping-error-rate---err_rate-)
         * [Rate of opposite homozygote errors](#rate-of-opposite-homozygote-errors---err_hom_rate-)
         * [Missingness rate](#missingness-rate---miss_rate-)
         * [Pseudo-haploid rate](#pseudo-haploid-rate---pseudo_hap-)
         * [Maintaining phase in output](#maintaining-phase-in-output---keep_phase)
         * [Listing input sample ids used as founders](#listing-input-sample-ids-used-as-founders---founder_ids)
         * [Retaining extra input samples](#retaining-extra-input-samples---retain_extra-)
   * [Extraneous tools](#extraneous-tools)
      * [Plotting pedigree structures: plot-fam.R](#plotting-pedigree-structures-plot-famr)

------------------------------------------------------

Basic usage:
------------

    ./ped-sim -d <in.def> -m <map file> -i <in.vcf/in.vcf.gz> -o <out_prefix> --intf <filename>

To use a non-interference crossover model, i.e., a Poisson model, use:

    ./ped-sim -d <in.def> -m <map file> -i <in.vcf/in.vcf.gz> -o <out_prefix> --pois

The simulator produces four output files: `[out_prefix].vcf` (or
`[out_prefix].vcf.gz`), `[out_prefix].bp`, `[out_prefix].fam`, and
`[out_prefix].log`. Descriptions of each of these input and output files are
below. Run `ped-sim` without arguments to see a full listing of options. This
document describes all options with the non-required options described at the
end.

<!--TODO: give example; want to make small VCF with 1kG samples for this-->

------------------------------------------------------

Compiling
---------

To compile the ped-sim program, most Linux/Unix-based users should simply be
able to type

    make

Other systems may require editing of the Makefile or alternate means of
compiling.

------------------------------------------------------

Def file
--------

The def file describes the pedigree structure(s) to be simulated. Comments are
allowed on a line by themselves beginning with #. Example def files are in the
`example` directory, and a description of the two example files is below.

The first line of a pedigree specification contains four columns:

    def [name] [#copies] [#generations] <sex of i1>

`[name]` gives the name of the pedigree, which must be unique for each pedigree
structure in a given simulation run (i.e., a given def file). The simulator
uses this to generate the simulated individuals' sample ids (details in
[Sample ids for simulated individuals](#samp-ids)).

`[#copies]` gives the number of replicate simulations of the given pedigree
structure to produce. While the replicates all have the same structure, they
will descend from different founders and will have different randomized sex
assignments (when using sex specific maps and assuming `<sex of i1>` is not
specified), and so are independent.

`[#generations]` indicates the number of generations in the pedigree.

`<sex of i1>` is an optional field giving the sex (F for female, M for male) of
the individual with id `i1` (the reproducing individual) in each branch. See
[Sample ids for simulated individuals](#samp-ids).

After this first line, the def file lists simulation details corresponding to
various generations in the pedigree. Each such line has the following format:

    [generation#] [#samples_to_print] <#branches> <parent_specifications>

`[generation#]` gives an integer value for the pedigree generation number. This
value can range from 1 (the earliest generation) to the total number of
generations included in the pedigree, as listed on the first line of the
specification (`[#generations]` just above).

`[#samples_to_print]` indicates how many samples the simulator should print for
each branch (defined more fully below) in the indicated generation; it defaults
to 0 so that no individuals are printed in generations that are not explicitly
listed. All individuals in a given branch and given generation have the same
parents and so are full siblings of one another. Because only one member of
each branch can have children, setting this to a value greater than 1 generates
data for individuals that do not have any offspring. **To simulate a pedigree in
which multiple full siblings each have children, increase the number of
branches in the third `<#branches>` field.** Note that any _founder_ spouses of
the person who does have children in a branch are always printed if this field
is greater than 0. These spouses do not count in the value of this field: the
value indicates how many full siblings to generate for each branch; if it is
non-zero, that number of siblings and any spouses will be printed. Also note
that if a branch contains a founder individual (such as in generation 1), it
will only ever contain one individual (and any spouses of that person): the
`[#samples_to_print]` value will only control whether (if it is greater than 0)
or not (if it is 0) that founder and his/her spouses are printed.

`<#branches>` is an optional field. By default:

* Generation 1 has one branch that contains a founder individual, and
generation 2 has two branches that are both children of the founder individual
and his/her spouse from generation 1; thus they are full siblings.
* Other than generations 1 and 2, every generation includes the same number of
branches as the previous generation. In consequence, not all generations need
an explicit listing in the def file.

For generation 1, multiple branches are allowed, and all such branches contain
only founder individuals. For all other generations, if the parent specification
(next field described below) is left empty, the parents of each branch are as
follows:

* If the number of branches is *an integer multiple* n *times the number of
branches in the previous generation*, individuals in each branch *i* in the
previous generation are the parents of branches *n\*(i-1)+1* through *n\*i* in
the current generation. (Thus, *if the number of branches is the same*,
individuals in each branch *i* in the previous generation are the parents of
branch *i* in the current generation.)
* If the number of branches is *less than the number of branches in the
previous generation*, individuals in each branch *i* in the previous generation
are the parents of branch *i* in the current generation. (Thus some branches in
the previous generation do not have children.)
* If the number of branches is *greater than but not divisible by the number of
branches in the previous generation*, branches 1 through *n\*p* have parents
assigned according to the integer multiple case just above; here *p* is the
number of branches in the previous generation and *n* is the largest integer
divisor by *p* of the number of branches in the current generation. The
remaining branches *n\*p+1* through *n\*p+r* contain founder individuals (as in
generation 1), where *r* is the remainder branch number after integer division.

The above are defaults, and the parents of a branch can be specified using the
parent specification entry.

`<parent_specifications>` is optional. The default parent assignments are given
above. The format of the specifications is any of:

    [current_branches]:
    [current_branches]:[parent_branch1]
    [current_branches]:[parent_branch1]_[parent_branch2]
    [current_branches]:[parent_branch1]_[parent_branch2]^[parent_branch2_generation]

Here `[current_branches]` contains a range of branches from the current
generation whose parents are specified after the `:` character. This can be
a single branch or comma separated list of branches such as `1,2,3` or, for a
contiguous range you can use a hyphen as in `1-3`. Any combination of
contiguous ranges and comma separated sets of branches are allowed such as
`2-5,7,9-10`.

If no text appears after the ':', the indicated branches will contain founder
individuals. For example, `1-3,5:` specifies that branches 1 through 3 and 5
should contain founders.

If only `[parent_branch1]` is listed, the reproducing parent from that
branch in the previous generation has children with a founder spouse. So for
example, `1,7:2` indicates that branches 1 and 7 will be the children of an
individual from branch 2 in the previous generation and a founder spouse.
Because these branches are listed together, they will contain full siblings.
To generate these branches as half-sibling children of branch 2 the
specification should be `1:2 7:2`. Here, branch 2 contains the parent of both
individuals, but the separate specifications for branches 1 and 7 ensures that
that parent has children with two different founder spouses, making the
children in the branches half-siblings.

If two parent branches are listed as in `[parent_branch1]_[parent_branch2]`,
the two reproducing parents are from the indicated branches in the previous
generation. Thus, for example, `2,4:1_3` indicates that branches 2 and 4 from
the current generation are to be the children of the reproducing parents in
branches 1 and 3 in the previous generation.

To have parents from different generations, the format is
`[parent_branch1]_[parent_branch2]^[parent_branch2_generation]`. Here, one
parent (the first one listed) is required to be in the previous generation and
the second parent comes from some other generation. Because the children are in
the current generation, the generation of both parents must be earlier than the
current one. As an example `2:1_3^2` indicates that branch 2 in the current
generation has parents from branch 1 in the previous generation and branch
3 from generation 2.

The simulator keeps track of the constraints on the sex of the parents implied
by the requested matings and will give an error if it is not possible to assign
sexes necessary to have offspring. For example, `1:1_3 2:1_4 3:3_4` is
impossible since the reproducing individuals in branches 3 and 4 must be the
same sex in order to both have children with the individual in branch 1.

### Example def file: `example/full_half_1st_2nd_cousins.def`

The first entry in the `example/full_half_1st_2nd_cousin.def` file simulates
a single pedigree that has four generations:

    def full1-2-cous 1 4
    3 0 4
    4 1

Because the first two generations are not explicitly listed, they have the
default number of branches: one and two for generations 1 and 2, respectively.
Since the number of samples to print is 0 by default, no samples are printed
from these generations.  In generation 3, there are four branches, with
generation 2, branch 1 a parent (along with a founder spouse) of branches 1 and
2, and generation 2, branch 2 a parent (with a founder spouse) of branches 3
and 4.  No samples from generation 3 are printed. Finally, generation 4 has
four branches, the same as the previous generation, with one sample printed per
branch, or a total of four individuals printed. Because the four branches in
generation 3 included two sets of full siblings, two pairs of the four samples
in generation 4 are first cousins. The other pairs are second cousins, and
their most recent common ancestors are in generation 1.

The second entry in this file is very similar to the first:

    def half1-2-cous 1 4
    2 0 2 1:1 2:1
    3 0 4
    4 1

The only difference between this pedigree and the one above is in generation 2.
This generation once again has two branches, and each branch has the
reproducing individual from generation 1, branch 1 as one of their parents.
However, because the specification is separated for the two branches and
includes only branch number 1, these branches are the offspring of two
different founders and are thus half-siblings. In consequence, the ultimate
descendants in generation 4 are a mix of (full) first cousins and half-second
cousins.

### Example def file: `example/second_deg.def`

The first entry in the `example/second_deg.def` file simulates 10 pedigrees
named `grandparent`, with data printed for two grandparents and one grandchild.

    def grandparent 10 3
    1 1
    2 0 1
    3 1

This indicates that the founder individual (and therefore his/her spouse) from
the one branch in generation 1 (note: default has one branch in generation 1)
should have data printed. Generation 2 has a default of two branches, but since
we only want one grandchild, we explicitly set this to one branch and do not
print individuals from that generation. Generation 3 prints one individual, and
it has only one branch since unspecified branch numbers are the same as the
previous generation and that previous generation (2) has only one branch.

The second entry simulates 10 pedigrees named `avuncular`:

    def avuncular 10 3
    2 2 1
    3 1

Here, generation 1 has the default of one branch with no data printed.
Generation 2 also has one branch and there are two full siblings in that branch.
Both of these individuals get printed along with the spouse of the reproducing
individual in that branch. Finally, generation 3 has one branch (since
generation 2 has that number) and one individual gets printed. Thus, for each
replicate pedigree, there is a pair of samples with an avuncular relationship
included in the output, along with two parent-offspring sample pairs.

The third entry simulates 10 pedigrees named `hs` for half-sibling:

    def hs 10 2
    2 1 2 1:1 2:1

Here, generation 1 has the default of one branch with no data printed.
Generation 2 has two branches. With the parent specification of `1:1 2:1`,
both these branches have the reproducing individual from branch 1 as a parent.
They are both also children of two distinct founders and are therefore
half-siblings. This prints two individuals per pedigree, one from each branch.

The last entry simulates 10 pedigrees named `dc` for double cousins:

    def dc 10 3
    1 0 2
    2 0 4
    3 1 2  1:1_3  2:2_4

Generation 1 has two branches, both containing founders. Generation 2 has four
branches: branches 1 and 2 are full sibling children of generation 1, branch 1;
branches 3 and 4 are also full siblings and the children of generation 1,
branch 2. In generation 3, there are only 2 branches: branch 1 contains the
child of individuals from generation 2, branches 1 and 3; branch 2 contains the
child of individuals from generation 2, branches 2 and 4. As the individuals in
branches 1 and 2 are full siblings and those in branches 3 and 4 are also full
siblings, the third generation samples are "double cousins." Only these
two double cousin individuals from the last generation are printed.

------------------------------------------------------

Map file <a name="map-file"></a>
--------

The genetic map file contains three columns for a sex-averaged map or four
columns if using male and female maps. The format of this file is:

    [chromosome] [physical_position] [map_position0] <map_position1>

The chromosomes are expected to be listed in the same order as they are in
the input VCF file, with the physical positions in increasing order. The
chromosome names must match the names in the input VCF file.

`[map_position0]` is genetic position in centiMorgans, and should either be the
sex-averaged genetic position if using only one map, or should be the male
genetic position if using two maps. When using only one map, the simulator
samples all crossovers from that one map and does not distinguish male and
female parents.

`<map_position1>` is likewise a genetic position in centiMorgans and should
correspond to the female genetic position if given.

A high resolution sex-specific genetic map is available [here](https://github.com/cbherer/Bherer_etal_SexualDimorphismRecombination),
and is described in [Bh�rer, et al. 2017](http://dx.doi.org/10.1038/ncomms14994).
To generate a map file in the format the simulator requires with both male and
female genetic positions, run the following bash commands:

    printf "#chr\tpos\tmale_cM\tfemale_cM\n" > refined_mf.simmap
    for chr in {1..22}; do
      paste male_chr$chr.txt female_chr$chr.txt | awk -v OFS="\t" 'NR > 1 && $2 == $6 {print $1,$2,$4,$8}' \
        | sed 's/^chr//' >> refined_mf.simmap;
    done

This generates a file called `refined_mf.simmap` that can be passed to the
simulator.

------------------------------------------------------

Input VCF file
--------------

All founders in the simulated pedigrees are randomly sampled individuals
from the input VCF file. This VCF must contain phased data for all individuals,
with no missing data for any site. As most phasers automatically impute missing
data, the latter requirement should be easily met. __At present, Ped-sim only
supports the GT FORMAT field, so please remove additional fields before using
Ped-sim.__

The input VCF file can be gzipped, and if it is, Ped-sim prints the output VCF
in gzipped format.

If your aim is to simulate only crossover break points and not genotype data,
you can use an empty file (including /dev/null) as the input VCF file.
Ped-sim will perform the simulation, generate a BP file, and then generate an
empty output VCF since the input contains no markers.

------------------------------------------------------

Crossover model
---------------

Ped-sim performs simulation from either of two crossover models: one that
incorporates crossover interference, or a Poisson model. When the necessary
parameters for crossover interference are available, we recommend using this
model, as it is motivated by biological data and produces quite different
results than a Poisson model. The two options for crossover models that Ped-sim
supports are below.

### Crossover interference model: `--intf <file>`

The `--intf <file>` option simulates from the [Housworth and Stahl 2003](http://www.cell.com/ajhg/fulltext/S0002-9297%2807%2963904-4)
model of crossover. This model requires specification of `nu` and `p`
parameters for each chromosome. The `interference` subdirectory in the
repository contains a file `nu_p_campbell.tsv` with estimates of these
parameters for the human autosomes from [Campbell et al.
2015](https://www.nature.com/articles/ncomms7260).

As with the VCF, the interference file must list chromosomes in the same order
as the genetic map, and the chromosome names must be identical to the genetic
map. The --intf file requires parameters to be given for both sexes and
requires a genetic map for both males and females. Ped-sim will print an error
when running with `--intf` if the genetic map only has one set of map
positions.

The format of the interference file is:

    [chromosome] [nu_0] [p_0] [nu_1] [p_1]

The `[nu_0]` and `[p_0]` parameters correspond to the first genetic map given
(see [Map file](#map-file)), which is assumed to be male, and the `[nu_1]`
and `[p_1]` parameters correspond to the second genetic map, which is assumed
to be female.

### Poisson crossover model: `--pois`

Use the `--pois` option to simulate using a Poisson crossover model.

------------------------------------------------------

Output VCF file
---------------

The output VCF contains the simulated individuals, including only those samples
requested to be printed in the def file. For any generation in which there is a
request to print one or more samples, the simulator prints any spouses in that
generation as well as the primary branch individuals. See below for a
description of the sample ids of the simulated individuals.

------------------------------------------------------

Output BP file
--------------

The break points (BP) file lists information about the haplotypes of each
sample. All founders have a unique numerical id for each of their two
haplotypes, starting from 0 and ranging to 2\**F*-1, where *F* is the number of
founders across all simulated pedigrees. Within the BP file, there are two
lines for every sample requested to be printed (according to the def file).
Each line begins with the sample id of the simulated individual (described
below), the sex of that person, either `s0` for male or `s1` for female, which
haplotype, `h0` or `h1`, that line describes, and then a variable number of
segments for each simulated chromosome.

For each simulated chromosome, there is starting physical position and one or
more break points. The start description is listed as

    [chromosome]:[start physical position]

Following this, break points where crossovers occurred are indicated as

    [founder haplotype]:[physical position]

The range of physical positions between the previous break point (or start
physical position for the first marker) descend from `[founder haplotype]`
number. For example, consider:

    grandparent2_g3-b1-i1 s0 h0 22|17178586 9:25639567 8:45864504 6:51039778

This line describes the haplotypes and break points inherited by an individual
with id `grandparent2_g3-b1-i1`. That individual is simulated as male (`s0`),
and the description is for their first haplotype (`h0`). Only chromosome 22 is
listed, and it begins at position `17178586`. Note that the start and end
positions -- the last break point position on any chromosome -- are dictated by
the input genetic map. The first break point `9:25639567` indicates that this
individual inherited haplotype 9 from position 17,178,586 through 25,639,567,
inclusive. The next break point `8:45864504` designates that the individual
inherited haplotype 8 from position 25,639,568 through 45,864,504. And the final
break point of `6:51039778` says that the individual received haplotype 6 from
position 45,864,505 through 51,039,778, the latter of which ends the chromosome.

------------------------------------------------------

Output fam file
---------------

The simulator produces a PLINK format fam file with the pedigree structures
simulated. This fam file contains all generated samples, including those that
are not requested to be printed in the def file. This enables the relationships
between all samples to be determined from the fam file alone.

------------------------------------------------------

Output log file
---------------

Information about the simulation run appears in the log file and is a copy of
what is printed to the console during execution. Notably this includes the
random seed used for a given simulation. Supplying the same input files with
the same random seed (assignable with the `--seed` option) will produce the
same simulation results.

------------------------------------------------------

Sample ids for simulated individuals <a name="samp-ids"></a>
------------------------------------

The simulated individuals' sample ids have the format
`[name][#]_g[#]-b[#]-i[#]`, or for spouses of reproducing individuals,
`[name][#]_g[#]-b[#]-s[#]`. Here, `[name]` is the pedigree name given in the
def file. The first number `[#]` is the copy number of the pedigree which
ranges from 1 to the number of copies of the given pedigree structure requested
in the def file (i.e., `[#copies]` above). The `g[#]` portion of the id gives
the generation number of the individual, which ranges from 1 to the total
number of generations in the pedigree. `b[#]` gives the branch number the
sample is contained in in the indicated generation; this ranges from 1 to the
total number of branches in that generation. Finally, `i[#]` gives the
individual number in the given branch and generation. This ranges from 1 to the
total number of samples requested to be simulated in the generation. Individual
`i1` is the reproducing individual that is the parent of any descendant
branches. When `i1` does have children, his/her founder spouses have the same
prefix id but end in `s[#]`, with the number ranging from 1 to the total number
of spouses of that individual. The number of spouses will only be 1 unless
parent specifications appear in the def file that indicate more founder spouses
should be used.

------------------------------------------------------

Extra notes: sex-specific maps
------------------------------

When simulating with sex-specific maps, it is necessary to include data for all
chromosomes in one run. This is because sex is assigned randomly, but only once
per run. Thus, to maintain consistency of the sex of each individual in a
given pedigree, all chromosomes need to be simulated in the same run.

------------------------------------------------------

Citing Ped-sim
--------------

If you use Ped-sim in your work, please cite [Caballero et al. (2019)](https://www.biorxiv.org/content/early/2019/01/22/527655).

------------------------------------------------------

Other optional arguments
------------------------

### Specifying random seed: `--seed <#>`

The `--seed <#>` option enables specification of the random seed to be used.
Without this option, the simulator generates a random seed using the current
time (including microseconds).

### Genotyping error rate: `--err_rate <#>`

To more accurately mimic real data, the simulator introduces genotyping errors
at a specified rate, defaulting to 1e-3. Set this value to 0 to keep the
allelic values identical to those in the founder haplotypes (from the input
data).

**Note: only pedigree samples have genotyping errors introduced;
`--retain_extra` samples maintain their original calls**

### Rate of opposite homozygote errors: `--err_hom_rate <#>`

SNP array genotype calling works by clustering allele intensities among a set
of samples. So if an individual is truly homozygous, its intensities are more
likely to fall in either the correct cluster or the heterozygous cluster, with
a lower probability of being called homozygous for the opposite allele.  While
we are unaware of a study that looks at error rates by "true" genotype class in
SNP array data, the `--err_hom_rate` option provides the ability to produce
different rates of errors for genotypes that are truly homozygous. The default
rate for generating an erroneous genotype that is homozygous for the opposite
alleles relative to the truth is 0, so errors at homozygous genotypes produce a heterozygote. If set to, say, .1, when a homozygous genotype is set to an
erroneous value, 10% of the time it is set homozygous for the opposite allele,
and 90% of the time is heterozygous. For equal rates of both these classes, set
the rate for this option to .5.

### Missingness rate: `--miss_rate <#>`

As real data includes missingness, the simulator introduces missing genotype
calls at a rate specified by this parameter, with a default of 1e-3. Set this
value to 0 for no missing genotypes.

Can only use `--miss_rate` or `--pseudo_hap` not both.

**Note: only pedigree samples have sites set to missing; `--retain_extra`
samples maintain their original calls**

### Pseudo-haploid rate: `--pseudo_hap <#>`

The `--pseudo_hap` option generates pseudo-haploid data with the rate of (fully)
missing data given by the argument (e.g., `--pseudo_hap .1` will have 10% of
sites completely missing data). Sites that do have data are all haploid for a
random allele sampled from the two original ones.

Can only use `--miss_rate` or `--pseudo_hap` not both.

**Note: only pedigree samples have sites set to missing or pseudo-haploid;
`--retain_extra` samples maintain their original calls**

### Maintaining phase in output: `--keep_phase`

By default the simulator produces a VCF that does not contain phase information.
The `--keep_phase` option will instead generate a VCF that maintains the
phase of all samples.

### Listing input sample ids used as founders: `--founder_ids`

Ped-sim assigns input samples as founders in the pedigrees it simulates. The
`--founder_ids` option prints a file called `[out_prefix].ids` that contains
two columns listing each founder sample id followed by the corresponding input
sample id Ped-sim assigned to that founder.

### Retaining extra input samples: `--retain_extra <#>`

The simulator uses samples from the input VCF as founder individuals, and will
exit if more founders are needed than available in the input VCF. If requested
using `--retain_extra`, the program will also print a specified number of input
samples that were not used as founders in the simulations. If the number is
less than 0 (e.g., `--retain_extra -1`), the simulator prints all unused input
samples. If the value is greater than 0, say 100, but fewer than this number of
unused samples exist, the simulator prints all the available samples.  When the
requested number to print is less than the number available, the simulator
randomly selects the samples to print from among all that were not used as
founders.

------------------------------------------------------

Extraneous tools
================

Plotting pedigree structures: `plot-fam.R`
------------------------------------------

The `plot-fam.R` script can be used to plot the pedigree structures produced by
`ped-sim` (or indeed for any PLINK format fam file). It requires the
[kinship2](https://cran.r-project.org/web/packages/kinship2/index.html)
R package and works by running

    ./plot-fam.R [base name]

This plots all pedigree structures given in the `[base name].fam` file. The
output files are named `[base name]-[family id].pdf`, with a file for each
family id (first column) in the fam file.

**Be mindful of the number of files this will produce:** it generates a pdf for
each *copy* of all the family structures in the file. It may be helpful to run
Ped-sim with the number of copies of each structure set to 1 when using this
script to check your structures.
