---
layout: page
title: BLABEL
subtitle: Skolemising Blank Nodes while Preserving Isomorphism
published: true
---


## Overview

**blabel** is a software package for labelling blank nodes in RDF graphs in a canonical manner. The package will label blank nodes in a manner that preserves [isomorphism](http://www.w3.org/TR/rdf11-concepts/#graph-isomorphism) and should be efficient for most real-world graphs. Options are provided to label blank nodes as blank nodes, or alternatively as IRIs, in the latter case effectively Skolemising them.

The new version of **blabel** (2016) also includes methods for leaning an RDF graph! This may be useful if you want to remove some redundancy from the RDF graph before you label the blank nodes, thus going further than isomorphism, preserving the [(simple) equivalence of RDF graphs](https://www.w3.org/TR/rdf11-mt/#simpleentailment), which are graphs that are not only structured the same (isomorphism) but in terms of simple semantics, say the same thing (equivalence).

Uses of the **blabel** package include (but are not limited to):

1. Leaning an RDF graph or testing if an RDF graph is lean.
2. [Skolemising blank nodes in an RDF graph](http://www.w3.org/TR/rdf11-concepts/#section-skolemization) such that the Skolems correspond for isomorphic RDF graphs (or equivalent RDF graphs with leaning enabled).
3. Computing a canonical hash/signature for an RDF graph that is unique modulo isomorphism (or modulo equivalence with leaning enabled).
4. Detecting isomorphic RDF graphs (or equivalent RDF graphs with leaning enabled) from a large set of graphs without requiring pairwise comparison.

## Code

Check out the code and other materials [here](https://github.com/aidhog/blabel/).

## Paper

More details about the algorithm used are available from the following paper:

Aidan Hogan. "[Skolemising Blank Nodes while Preserving Isomorphism](http://aidanhogan.com/docs/skolems_blank_nodes_www.pdf)". In the Proceedings of the 24th International World Wide Web Conference (WWW), Florence, Italy, May 18–22, 2015.

The paper also provides details of performance experiments which show that it should be efficient in practice.

## Command-line Use

In the `cl.uchile.dcc.blabel.cli` package, a main method called `LabelRDFGraph` is included to lean/label an RDF graph. Help can be found by calling the method without arguments or by entering `-h` as an argument.

The package accepts RDF in N-triples format from disk (`-i [filename]`) or from standard in (`-i std`). 

The labelled RDF graph will be written to a file (`-o [filename]`) or to standard out (`-o std`) as N-Triples. Input and output formats can be GZipped (`-igz` and/or `-ogz` respectively). 

The labelling relies on a hashing function where a selection of such can be chosen from (`-s 0|1|2|3|4`, where 0 is `MD5`, 1 is `MURMUR3_128`, 2 is `SHA1`, 3 is `SHA256`, and 4 is `SHA512`). 

One can also chose to output blank nodes (`-b`) or IRIs (default) to replace the input blank nodes, as well as a prefix to prepend to the label (e.g., a well-known IRI prefix; `-p [prefix]`). The onus is on the user to make sure that the prefix is properly escaped as a blank node/IRI to ensure valid output.

In the new version, one can now chose to lean the RDF graph before labelling (`-l`), or indeed can choose to just lean the RDF graph (`-lo`) and not re-label blank nodes.

There are two other options that are a little more complex to explain and relate to how the labels are scoped. An RDF graph can contain multiple blank node partitions with a partition for each connected set of blank nodes. By default, blank nodes are labelled uniquely for the input graph. But one can also choose to label uniquely per partition (`-upp`). Also, multiple isomorphic blank node partitions may exist in an RDF graph; by default these are distinguished, but one can choose to have these removed from the input (`-ddp`).

With respect to these last two options, consider the following RDF graph:

    _:a <p> _:b .
    _:b <p> _:c .
    _:c <p> _:a .
    _:x <p> _:y .  
    _:y <p> _:z .
    _:z <p> _:x .
    <u> <p> <v> .

The first three triples form one "partition" based on connected blank nodes, triples 4–6 form a second partition, and the last triple forms a ground partition.

The algorithm looks at each non-ground partition separately and will generate labels unique to that partition modulo isomorphism. Thus initially, if two non-ground partitions are isomorphic (i.e., they can be made equal by a one-to-one relabelling of blank nodes), they will have the same blank node labels. Otherwise they will share no labels.

The default behaviour (without setting `-upp` or `-ddp`) is that if two or more isomorphic partitions are found, they are distinguished by hashing in a sequential ID to identify each partition. As a final step, a hash of the entire graph (including the ground information) is computed and muxed with all the blank node labels. Thus the labels computed are unique to the entire graph modulo isomorphism.

If `-dpp` is set, then the partitions are not distinguished, the labels computed are the same, and only one copy of each isomorphic partition is preserved. In effect, the output would be the same for the above graph with or without the first three triples if `-dpp` is set.

If `-upp` is set, the entire graph is not hashed, rather labels are unique to the partition. For example, the final ground triple will not affect the labels of the blank nodes if `-upp` is set. In fact, no data outside the partition will affect the labels of blank nodes in that partition.

So `-dpp` removes isomorphic partitions and `-upp` labels blank nodes on the scope of a partition rather than a graph.

Which you choose depends on the scenario. Hopefully if you understand the options, you will know which one is best for you. The default (no `-upp` or `-dpp`) will preserve the isomorphism of the input graph.

Note that if you chose to lean beforehand with `-l`, the second isomorphic partition would have been removed beforehand anyways. Leaning is applied separately before any of the above considerations.

An example argument list for the class might be:

     -i input.nt.gz -igz -o output.nt.gz -ogz -s 3 -l -p http://example.org/.well-known/genid/

This gives the input file, the output file, states that the input is GZipped, the output should be GZipped, to use `SHA256`, to lean beforehand, and to append the labels to a Skolem IRI prefix in a form recommended by [RDF 1.1](http://www.w3.org/TR/rdf11-concepts/#section-skolemization).

## Software re-use

If you wish to include **blabel** as a package in your code, I would advise to check the `LabelRDFGraph` class for an example on how to initialise, envoke and get the results from the `GraphLabelling` class or `GraphLeaning` classes. 

In general, the default settings are most suitable in most cases. In terms of leaning, the `DFSGraphLeaning` class is recommended over `BFSGraphLeaning`. 

If you are hoping to integrate this code with Jena, please see the `cl.uchile.dcc.blabel.jena` package, where you'll find some commented out code to take a Jena Model instance and convert it to an input NxParser Iterator assumed by this package (you should load the objects from that iterator into some collection, which you can then pass to the class). It's commented out because to include the Jena libraries would increase the dependencies tenfold.

## Known limitations

* The package assumes that the RDF graph in question can be loaded into memory and shallow copied a few times. How many times depends on the complexity of the underlying structure.
* Leaning and labelling algorithms are exponential. However, exponential cases are very unlikely to occur in practice (see the paper above for performance details; for example, it can compute a labelling on a clique with k=55 in under 10 minutes; real-world graphs are seemingly unlikely to be complicated or uniform enough to cause real trouble).
* The algorithm uses hashing. Hash collisions can happen. Assuming an ideal hashing function, at 128 bits, it's unlikely to be an issue in practice (see the paper for more details). However, some of the hashing functions are less ideal than others and for large and very uniform graphs, hash collisions can occur. The code detects when this happens and in general will silently recover in a deterministic manner for a given graph, but across graphs, false hash collisions *may* occur in rare/strange/really very unlucky cases.
* If you want the results to be consistent across multiple instances, of course you will have to use the same settings and possibly even the same version! At the moment, there is no guarantee that different versions will produce, e.g., the same Skolem IRIs.
* The package is based on the [nxparser](https://code.google.com/p/nxparser/) package, which is a homebrew parser for N-Triples and related formats. The parser is designed to be simple and fast, but may not compliant in some minor respects. It does not validate input nor output and may not be up-to-date with changes in RDF 1.1.
* I am not a software engineer. This is research code. I tried to keep it fairly clean in the interest of making my own life easy but modern conveniences such as unit tests or Maven are not included in the package. Likewise there might be bugs in the command-line interface (which I've tested a bit but not thoroughly). At the same time, in `cl.uchile.dcc.blabel.test`, you'll find a class `TestFramework` that does a bunch of internal consistency checks, and I've run this over lots of graphs (both real-world and synthetic) and not found anything problematic.
* I can only offer limited support for the package. Generally it is offered as is, but please if you do find issues, log them in the issue tracker!

## Contributions

If you would like to contribute something to the package, let me know! There's a couple of things I'd love to see done but don't have the bandwidth/expertise to jump into myself:

* Maven-ising the package and dependencies
* Better integration with Jena and/or Sesame
* Maybe better documentation with JavaDoc

I guess it all depends on how much demand there is for something like **blabel**. :)

Also thanks to Emir Muñoz for starting this Jekyll page!!

## Licence and Dependencies

The package is available under an Apache Licence 2.0. The package also uses three external packages:

* Apache CLI (Apache License 2.0) for Command Line Interfaces.
* Guava (Apache License 2.0) for hashing implementations.
* NxParser (New BSD License) for parsing and data model.

## Reproducing results

Please see [here](http://aidanhogan.com/skolem/) for a static version of the code used for the paper above, as well as a pointer to the graphs used and instructions on how experiments can be reproduced. If you are looking for reproducibility details for the extended journal paper under review, please see [here](http://aidanhogan.com/blabel/index.html).

Some of the graphs used in the above experiments were taken from a [standard graph isomorphism benchmark for "Bliss"](http://www.tcs.hut.fi/Software/bliss/benchmarks/index.shtml) and converted to RDF. We also used [BTC 14](http://km.aifb.kit.edu/projects/btc-2014/) for testing.

## Happy blabelling
