![alt text](../img/banner.jpg "banner")
# 4.1 Core Components <a id="corecomponents"></a>

<a id="eve"></a>
### eve
***
eve takes a list of event ids in binary format as input and generates a partition of event ids as a binary data stream, according to the parameters supplied. Events are "shuffled" by being assigned to processes one by one cyclically, rather than being distributed to processes in blocks, in the order they are input. This helps to even the workload by process, in case all the "big" events are together in the same range of the event list.

##### Output stream
The output stream is a simple list of event_ids (4 byte integers).

##### Parameters
Required parameters are;
* process number - specifies the process number to receive a partition of events
* number of processes - determines the total number of partitions of events to distribute to processes

Optional parameters are;
* -n - No shuffled events. Events are split up and distributed in blocks in respect of the order they are input.


##### Usage
```
$ eve [parameters] > [output].bin
$ eve [parameters] | getmodel | gulcalc [parameters] > [stdout].bin
```

##### Example
```
$ eve 1 2 > events1_2_shuffled.bin 
$ eve -n 1 2 > events1_2_unshuffled.bin 
$ eve 1 2 | getmodel | gulcalc -r -S100 -i - > gulcalc1_2.bin
```

In this example, the events from the file events.bin will be read into memory and the first half (partition 1 of 2) would be streamed out to binary file, or downstream to a single process calculation workflow.

##### Internal data
The program requires an event binary. The file is picked up from the input sub-directory relative to where the program is invoked and has the following filename;
* input/events.bin

The data structure of events.bin is a simple list of event ids (4 byte integers).

[Return to top](#corecomponents)

<a id="getmodel"></a>
### getmodel 
***
getmodel generates a stream of effective damageability distributions (cdfs) from an input list of events. Specifically, it combines the probability distributions from the model files, footprint.bin and vulnerability.bin, to generate effective damageability cdfs for the subset of exposures contained in the items.bin file and converts them into a binary stream. 

This is reference example of the class of programs which generates the damage distributions for an event set and streams them into memory. It is envisaged that model developers who wish to use the toolkit as a back-end calculator of their existing platforms can write their own version of getmodel, reading in their own source data and converting it into the standard output stream. As long as the standard input and output structures are adhered to, each program can be written in any language and read any input data.

##### Output stream

| Byte 1 | Bytes 2-4 |  Description                                   |
|:-------|-----------|:-----------------------------------------------|
|    0   |     1     |  cdf stream                                    |

##### Parameters
None

##### Usage
```
$ [stdin component] | getmodel | [stout component]
$ [stdin component] | getmodel > [stdout].bin
$ getmodel < [stdin].bin > [stdout].bin
```

##### Example
```
$ eve 1 1 | getmodel | gulcalc -r -S100 -i gulcalci.bin
$ eve 1 1 | getmodel > getmodel.bin
$ getmodel < events.bin > getmodel.bin 
```

##### Internal data
The program requires the footprint binary and index file for the model, the vulnerability binary model file, and the items file representing the user's exposures. The files are picked up from sub-directories relative to where the program is invoked, as follows;

* static/footprint.bin
* static/footprint.idx
* static/vulnerability.bin
* static/damage_bin_dict.bin
* input/items.bin

The getmodel output stream is ordered by event and streamed out in blocks for each event. 

##### Calculation
The program filters the footprint binary file for all 'areaperil_id's which appear in the items file. This selects the event footprints that affect the exposures on the basis on their location.  Similarly the program filters the vulnerability file for 'vulnerability_id's that appear in the items file. This selects damage distributions which are relevant for the exposures. Then the intensity distributions from the footprint file and conditional damage distributions from the vulnerability file are convolved for every combination of areaperil_id and vulnerability_id in the items file. The effective damage probabilities are calculated by a sum product of conditional damage probabilities with intensity probabilities for each event, areaperil, vulnerability combination and each damage bin.  The resulting discrete probability distributions are converted into discrete cumulative distribution functions 'cdfs'.  Finally, the damage bin mid-point from the damage bin dictionary ('interpolation' field) is read in as a new field in the cdf stream as 'bin_mean'.  This field is the conditional mean damage for the bin and it is used to facilitate the calculation of mean and standard deviation in the gulcalc component. 

[Return to top](#corecomponents)

<a id="gulcalc"></a>
### gulcalc
***
The gulcalc program performs Monte Carlo sampling of ground up loss using interpolation of uniform random numbers against the effective damage cdf, and calculates mean and standard deviation by numerical integration of the cdfs. The sampling methodology has been extended beyond linear interpolation to include point value sampling and quadratic interpolation. This supports damage bin intervals which represent a single discrete damage value, and damage distributions with cdfs that are described by a piecewise quadratic function. 

Gulcalc also performs back-allocation of total coverage losses to the contributing subperil item losses (for multi-subperil models).  This occurs when there are two or more items representing losses from different subperils to the same coverage, such as wind loss and storm surge loss, for example. In these cases, because the subperil losses are generated independently from each other it is possible to result in a total damage ratio of > 1 for the coverage, or a total loss of > TIV. gulcalc provides the following options for summing and back-allocating the total coverage ground up loss;

| Allocation option | Description                                                                                         | 
|:------------------|:----------------------------------------------------------------------------------------------------|
| 1                 | Sum damage ratios and cap to 1. Back-allocate in proportion to contributing subperil damage ratio   |

##### Stream output

| Byte 1 | Bytes 2-4 |  Description                                   |
|:-------|-----------|:-----------------------------------------------|
|    1   |     1     |  gulcalc item stream **(deprecated)**          |
|    1   |     2     |  gulcalc coverage stream **(deprecated)**      |
|    2   |     1     |  loss stream                                   |

##### Parameters
Required parameters are;
* -S{number}. Number of samples
* -a{number} -i{destination} . Specifies stdout option 2/1 loss stream with an allocation option
**or**
* -c and/or -i {destination}. Specifies stdout option 1/1 gulcalc coverage or 1/2 gulcalc item stream respectively. 

The destination is either a filename or named pipe, or use - for standard output.

Optional parameters are;
* -L{number} loss threshold (optional) excludes losses below the threshold from the output stream
* -d debug mode - output random numbers rather than losses (optional)
* -A automatically hashed seed driven random number generation (default)
* -R{number} Number of random numbers to generate dynamically 
* -s{number} Manual seed for random numbers (to make them repeatable)
* -r Read random numbers from random.bin file

##### Usage
```
$ [stdin component] | gulcalc [parameters] | [stout component]
$ [stdin component] | gulcalc [parameters]
$ gulcalc [parameters] < [stdin].bin 
```

##### Example
```
'To stdout loss stream
$ eve 1 1 | getmodel | gulcalc -R1000000 -S100 -a1 -i - | fmcalc > fmcalc.bin
$ eve 1 1 | getmodel | gulcalc -R1000000 -S100 -a1 -i - | summarycalc -i -1 summarycalc1.bin
$ eve 1 1 | getmodel | gulcalc -r -S100 -a1 -i gulcalci.bin
$ gulcalc -r -S100 -i -a1 gulcalci.bin < getmodel.bin 
'To stdout gulcalc coverage and item stream (deprecated)
$ eve 1 1 | getmodel | gulcalc -R1000000 -S100 -c - | summarycalc -g -1 summarycalc1.bin
$ eve 1 1 | getmodel | gulcalc -R1000000 -S100 -i - | fmcalc > fmcalc.bin
$ eve 1 1 | getmodel | gulcalc -r -S100 -i gulcalci.bin -c gulcalcc.bin
$ gulcalc -r -S100 -i gulcalci.bin -c gulcalcc.bin < getmodel.bin 
```

##### Internal data
The program requires the damage bin dictionary binary for the static folder and the item and coverage binaries from the input folder. The files are found in the following locations relative to the working directory with the filenames;

* static/damage_bin_dictionary.bin
* input/items.bin
* input/coverages.bin

If the user specifies -r as a parameter, then the program also picks up a random number file from the static directory. The filename is;
* static/random.bin

##### Calculation
The stdin stream is a block of cdfs which are ordered by event_id, areaperil_id, vulnerability_id and bin_index asccending, from getmodel. The gulcalc program constructs a cdf for each item, based on matching the areaperil_id and vulnerability_id from the stdin and the item file.

For each item cdf and for the number of samples specified, the program draws a random number and uses it to sample ground up loss from the cdf using one of three methods, as follows;

For a given damage interval corresponding to a cumulative probability interval that each random number falls within;
* If the conditional mean damage (of the cdf) is the mid-point of the damage bin interval (of the damage bin dictionary) then the gulcalc program performs linear interpolation. 
* If the conditional mean damage is equal to the lower and upper damage threshold of the damage bin interval (i.e the bin represents a damage value, not a range) then that value is sampled.
* Else, the gulcalc program performs quadrative interpolation using the bin_mean to calculate the quadratic equation in the damage interval.

An example of the three cases and methods is given below;
 
| bin_from | bin_to |  bin_mean | Method used             |
|:---------|--------|-----------| :-----------------------|
|    0.1   |  0.2   |    0.15   | Linear interpolation    |
|    0.1   |  0.1   |    0.1    | Sample bin value        |
|    0.1   |  0.2   |    0.14   | Quadratic interpolation |

If the -R parameter is used along with a specified number of random numbers then random numbers used for sampling are generated on the fly for each event and group of items which have a common group_id using the Mersenne twister psuedo random number generator (the default RNG of the C++ v11 compiler).  These random numbers are not repeatable, unless a seed is also specified (-s{number}).

If the -r parameter is used, gulcalc reads a random number from the provided random number file, which produces repeatable results. 

The default random number behaviour (no additional parameters) is to generate random numbers from a seed determined by a combination of the event_id and group_id, which produces repeatable results. See [Random Numbers](RandomNumbers.md) for more details.

Each sampled damage is multiplied by the item TIV, looked up from the coverage file.

The mean and standard deviation of ground up loss is also calculated. For each cdf, the mean and standard deviation of damage is calculated by numerical integration of the effective damageability probability distribution and the result is multiplied by the TIV. The results are included in the output to the stdout stream as sidx=-1 (mean) and sidx=-2 (standard deviation), for each event and item (or coverage).

At this stage, there are three possible output streams, as follows;

* stream 1/1: If the -i parameter is specified, then the ground up losses for the items are output to the gulcalc item stream.
* stream 1/2: If the -c parameter is specified, then the ground up losses for the items are grouped by coverage_id and capped to the coverage TIV, and the ground up losses for the coverages are output to the stream.
* stream 2/1: If the -i and -a{} parameters are specified, then the ground up losses for the items are capped by coverage and back-allocated to items using the allocation method specified by the number provided to parameter a, and output to the loss stream.

Stream 2/1 also contains a loss sample for sidx -3, which is the 100% damage scenario for all items with the specified allocation method applied to it.

[Return to top](#corecomponents)

<a id="fmcalc"></a>
### fmcalc
***
fmcalc is the reference implementation of the Oasis Financial Module. It applies policy terms and conditions to the ground up losses and produces loss sample output.  It reads in losses from gulcalc at item level (gulcalc item stream 1/1 or loss stream 2/1) and it can run recursively on losses output from fmcalc (loss stream 2/1) and apply several consecutive sets of policy terms and conditions. 

##### Stream output

| Byte 1 | Bytes 2-4 |  Description                                   |
|:-------|-----------|:-----------------------------------------------|
|    2   |     1     |  loss stream                                   |

##### Parameters
Optional parameters are;
* -p {string} The location of the input files. The default location is the 'input' directory in the present working directory
* -a{integer} The back allocation rule to apply. The options are 0 (no allocation), 1 (ground up loss basis) or 2 (prior level loss basis). The default is 0. 
* -n Output net losses. Net losses are the difference between the input loss and the calculated loss. The default is to output the calculated loss.

##### Usage
```
$ [stdin component] | fmcalc [parameters] | [stout component]
$ [stdin component] | fmcalc [parameters] > [stdout].bin
$ fmcalc [parameters] < [stdin].bin > [stdout].bin
```

##### Example
```
$ eve 1 1 | getmodel | gulcalc -r -S100 -a1 -i - | fmcalc -p direct -a2 | summarycalc -f -2 - | eltcalc > elt.csv
$ eve 1 1 | getmodel | gulcalc -r -S100 -a1 -i - | fmcalc -p direct -a1 > fmcalc.bin
$ fmcalc -p ri1 -a2 -n < gulcalci.bin > fmcalc.bin
$ fmcalc -p direct | fmcalc -p ri1 -n | fmcalc -p ri2 -n < gulcalci.bin > fm_ri2_net.bin 
```

##### Internal data
For the gulcalc item stream input, the program requires the item, coverage and fm input data files, which are Oasis abstract data objects that describe an insurance or reinsurance programme. This data is picked up from the following files relative to the working directory by default;

* input/items.bin
* input/coverages.bin
* input/fm_programme.bin
* input/fm_policytc.bin
* input/fm_profile.bin
* input/fm_xref.bin

For loss stream input from either gulcalc or fmcalc, the program requires only the four fm input data files, 

* input/fm_programme.bin
* input/fm_policytc.bin
* input/fm_profile.bin
* input/fm_xref.bin

The location of the files can be changed by using the -p parameter followed by the path location relative to the present working directory. eg -p ri1

##### Calculation
See [Financial Module](FinancialModule.md)

[Return to top](#corecomponents)

<a id="summarycalc"></a>
### summarycalc 
***
The purpose of summarycalc is firstly to aggregate the samples of loss to a level of interest for reporting, thereby reducing the volume of data in the stream. This is a generic first step which precedes all of the downstream output calculations.  Secondly, it unifies the formats of the gulcalc and fmcalc streams, so that they are transformed into an identical stream type for downstream outputs. Finally, it can generate up to 10 summary level outputs in one go, creating multiple output streams or files.

The output is similar to the gulcalc or fmcalc input which are losses are by sample index and by event, but the ground up or (re)insurance loss input losses are grouped to an abstract level represented by a summary_id.  The relationship between the input identifier and the summary_id are defined in cross reference files called **gulsummaryxref** and **fmsummaryxref**.

##### Stream output

| Byte 1 | Bytes 2-4 |  Description                                   |
|:-------|-----------|:-----------------------------------------------|
|    3   |     1     |  summary stream                                |

##### Parameters

The input stream should be identified explicitly as -g for gulcalc item stream, -i for gulcalc loss stream or -f for fmcalc loss stream.

summarycalc supports up to 10 concurrent outputs.  This is achieved by explictly directing each output to a named pipe, file, or to standard output.  

For each output stream, the following tuple of parameters must be specified for at least one summary set;

* summaryset_id number. valid values are -0 to -9
* destination pipe or file. Use either - for standard output, or {name} for a named pipe or file.

For example the following parameter choices are valid;
```
$ summarycalc -i -1 -                                       
'outputs results for summaryset 1 to standard output
$ summarycalc -i -1 summarycalc1.bin                        
'outputs results for summaryset 1 to a file (or named pipe)
$ summarycalc -i -1 summarycalc1.bin -2 summarycalc2.bin    
'outputs results for summaryset 1 and 2 to a file (or named pipe)
```
Note that the summaryset_id relates to a summaryset_id in the required input data file **gulsummaryxref.bin** or **fmsummaryxref.bin** for a gulcalc input stream or a fmcalc input stream, respectively, and represents a user specified summary reporting level. For example summaryset_id = 1 represents portfolio level, summaryset_id = 2 represents zipcode level and summaryset_id 3 represents site level.

##### Usage
```
$ [stdin component] | summarycalc [parameters] | [stdout component]
$ [stdin component] | summarycalc [parameters]
$ summarycalc [parameters] < [stdin].bin
```

##### Example

```
$ eve 1 1 | getmodel | gulcalc -r -S100 -a1 -i - | summarycalc -i -1 - | eltcalc > eltcalc.csv
$ eve 1 1 | getmodel | gulcalc -r -S100 -a1 -i - | summarycalc -i -1 gulsummarycalc.bin 
$ eve 1 1 | getmodel | gulcalc -r -S100 -a1 -i - | fmcalc | summarycalc -f -1 fmsummarycalc.bin 
$ summarycalc -f -1 fmsummarycalc.bin < fmcalc.bin
```

##### Internal data
The program requires the gulsummaryxref file for gulcalc input (-g or -i option), or the fmsummaryxref file for fmcalc input (-f option). This data is picked up from the following files relative to the working directory;

* input/gulsummaryxref.bin
* input/fmsummaryxref.bin

##### Calculation
summarycalc takes either ground up loss (by coverage_id -g or by item_id -i) or insured loss samples (by output_id -f) as input and aggregates them to a user-defined summary reporting level. The output is similar to the input which are losses are by sample index and by event, but the ground up or insured losses are grouped to an abstract level represented by a summary_id.  The relationship between the input identifier, item_id or coverage_id for gulcalc or output_id for fmcalc, and the summary_id are defined in the internal data files.

For gulcalc coverage stream (1/2) input, summarycalc also calculates the maximum exposure value by summary_id and event_id. This is carried through in the stream header and output by eltcalc, aalcalc and pltcalc. For gulcalc coverage stream this is the sum of TIV for all coverages which intersect with a given event footprint on the basis of areaperil_id (regardless or not of whether there are any sampled losses). For gulcalc or fmcalc loss stream (2/1) input , this is the sum of the losses for sample index -3.  Sample index -3 is the 100% loss scenario for all impacted items passed through the financial module calculations.  Because the event footprints vary geographically, the value can be different for each event.

[Return to top](#corecomponents)

[Go to 4.2 Output Components section](OutputComponents.md)

[Back to Contents](Contents.md)
