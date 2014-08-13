---
layout: post
title: "R package for working with RRD files"
date: 2014-08-09 17:50:28 +0200
comments: true
categories:
- GSoC
- Code
---

This summer I had the immense pleasure to participate in the 10th edition of [Google Summer of Code](https://www.google-melange.com/gsoc/homepage/google/gsoc2014) as one of the students working for [Ganglia](http://ganglia.sourceforge.net/). 
I was mentored by [Daniel Pocock](http://danielpocock.com/) and my project dealt mostly with getting [RRD](http://en.wikipedia.org/wiki/RRDtool) data into [R](http://www.r-project.org/) to allow for efficient data analysis.

Tools such as [RRDtool](http://oss.oetiker.ch/rrdtool/) and [rrd2csv](https://code.google.com/p/rrd2csv/) can export an RRD file to xml or csv. 
While these formats can be imported into R, a more efficient and scalable approach would be to import the binary data directly (without exporting to an intermediate format first). 
To achieve this I implemented an [R package](https://github.com/pldimitrov/Rrd) that uses [librrd](http://oss.oetiker.ch/rrdtool/doc/librrd.en.html) to read  binary data from an RRD file directly into data structures in R.

The [.Call](http://www.biostat.jhsph.edu/~bcaffo/statcomp/files/dotCall.pdf) interface allows for calling C functions from R. 
Using .Call and the R headers for C makes it possible to pass R (_SEXP_) objects to C functions, generate, manipulate and return such back to R. 
A _SEXP_ object can be of any data type used in R - e.g. vector, _data.frame_, integer, etc.
This lets us implement C functions that take _SEXP_ objects as arguments, use core librrd functions (_rrd_fetch_r_, _rrd_info_, _rrd_first_, _rrd_last_, etc.) to retrieve data from RRD files, generate  _SEXP_ objects (such as _data.frame_-s), populate them with the retrieved data and return them to the caller function in R.


##The package provides the following functions:

- ***importRRD(filename, consolidation function, start, end, step)***

Acts as a wrapper around [_rrd_fetch_r_](https://github.com/oetiker/rrdtool-1.x/blob/master/src/rrd_fetch.c).
The user needs to have knowledge about the contents of the file beforehand to pick the right parameters.
Returns a _data.frame_ object containing the desired portion of the RRA that __best__ matches the parameters.
The data source names are retrieved and the columns of the _data.frame_ are named accordingly.
Due to the implementation specifics of _rrd_fetch_r_ the result does not include the row with timestamp _start_.


For more information on _rrd_fetch_r_, please consult the [official documentation](http://oss.oetiker.ch/rrdtool/doc/rrdfetch.en.html)



- ***importRRD(filename)***

Reads the metadata provided by _rrd_info_ and uses it to import all RRA-s in their entirety.
Retrieves an _rrd_info_t_ struct which contains the parameters for each RRA and uses these, together with the boundary values provided by _rrd_first_ and _rrd_last_ to fetch each RRA using _rrd_fetch_r_.
The user does not need to have any knowledge about the contents of the RRD file beforehand.
Returns a list of _data.frame_ objects, each labeled accordingly ("AVERAGE15" corresponds to an RRA with _consolidation function_ "AVERAGE" and _step_ 15).


- ***getVal(filename, consolidation function, step, timestamp)***

When the user is interested in looking at individual values in an RRA it might not be convenient to retrieve the entire contents of an RRD if the file is too large.
Retrieving portions of a specific RRA might result in a lot of reads from the RRD file (if few values are requested at a time) and indexing by row name in a _data.frame_ (if more values are requested) is known to be inefficient in R.


_getVal_ is optimized for working with indivudial values and uses a package-wide read-ahead cache to minimize the frequency of file reads.
The cache is implemented as an _environment_ object.
Environments in R can be used as hash tables as is demonstrated [here](http://broadcast.oreilly.com/2010/03/lookup-performance-in-r.html).
A key is generated from  _filename_, _consolidation function_ and _step_.
A _data.frame_ is associated with each key and represents a per-RRA cache.
The read-ahead size and the total size of the cache (per RRA) can be adjusted via setting the _rrd.cacheBlock_ and _rrd.cacheSize_ constants to the appropriate values.


Since looking up rows by row names (if we are to use _dataFrame["timestamp", ]_) is not done in constant time in R, the row index is calculated from the boundary timestamp values in the current cache and the _timestamp_ and _step_ parameters to allow for immediate indexing, instead.
In order for this to work we have to make sure the cache contains no gaps - i.e. the time distance between any two adjacent values in the cache is _step_.
_getVal_ also checks if the _timestamp_ is valid by looking at the boundaries of the cache and _step_. It will not cause an unnecessary call to rrd_fetch_r if it isn't.


The cache is updated as the following:

-- if _rrd.cache[[key]]_ is NULL (i.e. there are no entires in the cache for that RRA) - retrieve a timestamp range that includes the _rrd.cacheBlock_ next and previous values and store that in the cache

-- if the requested _timestamp_ is larger than the latest currently stored timestamp in the cache, extend the cache to also include the values up until _rrd.cacheBlock_ values after the requested one

-- similarly, extend the cache to include all values starting _rrd.cacheBlock_ ones before the requested _timestamp_ if it is smaller than the earliest currently stored

-- if the newly obtained cache entry for a given RRA is to exceed _rra.cacheSize_ in size, that entry is replaced with the  _rrd.cacheBlock_ next and previous values around the requested _timestamp_ as in the beginning

Another complication is due to the fact that, as mentioned above, _rrd_fetch_r_ will try to always find the RRA that matches the parameters best.
In case a value with a _timestamp_ smaller than the earliest available for the RRA of interest is requested, _rra_fetch_r_ will sometimes try to deliver a portion from another RRA in this RRD that contains a value with the specified _timestamp_. 
While this might be useful in certain situations, we really want to avoid this when working with the cache as each RRA we are getting values from is associated with a specific _data.frame_ in the cache and we want all values to be _step_ apart (the RRA that contains the _timestamp_ could have a different _step_).

In order to solve this problem, _getVal_ would obtain the earliest timestamp on the first cache-miss for a given RRA (and store it in a _rrd.first_ environment object). It will then not allow _rrd_fetch_r_ to be called with a _timestamp_ smaller then the first one for a given RRA.



##Example use

Retrieving a portion from a certain RRA in a RRD file and plotting the data.

{% codeblock %}

> rraPortion = importRRD("/tmp/bytes_in.rrd", "AVERAGE", 1401920280, 1401942000, 60)

> head(rraPortion)
            timestamp      sum num
1401920340 1401920340  24.0400   1
1401920400 1401920400  24.0400   1
1401920460 1401920460  24.0400   1
1401920520 1401920520 248.5335   1
1401920580 1401920580 432.2100   1
1401920640 1401920640 432.2100   1

> plot(rraPortion$timestamp, rraPortion$sum)

{% endcodeblock %}



{% img center /images/rplot.png plotting the RRA data in R %}


Retrieving a selection of values at specific timestamps from a certain RRA.

{% codeblock %}
> indices = c(1401941100, 1401941880, 1401939960, 1401933840, 1401931140, 1401941820, 1401922200)
> selection = do.call("rbind", lapply(indices, getVal, filename = "/tmp/bytes_in.rrd", cf="AVERAGE", step=60))
> selection
            timestamp       sum num
1401941100 1401941100  326.3180   1
1401941880 1401941880  710.8600   1
1401939960 1401939960  315.0500   1
1401933840 1401933840  395.6060   1
1401931140 1401931140  801.9072   1
1401941820 1401941820 2827.4305   1
1401922200 1401922200  822.5800   1

> plot(selection$timestamp, selection$sum)

{% endcodeblock %}

{% img center /images/rselectionplot.png plotting the RRA selection in R %}

The same result can be achieved with:


{% codeblock  %}
> rrd = importRRD("/tmp/bytes_in.rrd")
> selection = rrd$AVERAGE60[as.character(indices), ]
> plot(selection$timestamp, selection$sum)

{% endcodeblock %}

by importing the entire RRD file first.


One can take advantage of the many possibilities to manipulate _data.frame_ objects in R - e.g. performing various types of joins.

{% codeblock %}
> innerJoin = merge(rra1, rra2, by = "timestamp")
> head(innerJoin)

   timestamp       sum.x num.x       sum.y     num.y
1 1401921000    675.5600     1    458.6034 1.0000000
2 1401921600   6811.4233     1   1438.8657 1.0000000
3 1401922200    822.5800     1  30883.0741 1.0000000
4 1401922800   1677.4500     1   1462.9487 1.0000000
5 1401923400    433.0000     1   3849.4176 1.0000000
6 1401924000  24278.2883     1   3425.4432 1.0000000


{% endcodeblock %}

This makes it possible to easily prepare metrics data from multiple data sources and RRD files to be used for statistical analysis or fed to a neural network.

The join_all function from the [plyr](http://i.imgur.com/3ViWYSh.jpg) package can be used to join an arbitrary number of _data.frame_ objects

