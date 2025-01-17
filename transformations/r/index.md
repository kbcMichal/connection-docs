---
title: R Transformation
permalink: /transformations/r/
redirect_from:
    - /manipulation/transformations/r/

---

* TOC
{:toc}

[R](https://www.r-project.org/about.html) is designed for **advanced statistical computations**.
Apart from ready-to-use implementations of state-of-the-art algorithms, R's other great assets are vector and matrix
computations. R transformations complement Python and SQL transformations where computations or
other operations are too difficult. Common data operations like joining, sorting, and grouping, however, are still
easier and faster to do in [SQL Transformations](/transformations/).

## Environment
The R script is running in an isolated [Docker environment](https://developers.keboola.com/integrate/docker-bundle/).
The current R version is R **4.0.5**. The R version is updated regularly, few weeks after the official release.
The update is always announced on the [status page](http://status.keboola.com/).

When we update the R version, we offer --- for a limited time --- the option to switch to the previous version. You can
switch the version in the transformation detail by clicking on the `Change Backend` label:

{: .image-popup}
![Screenshot - Transformations Versions](/transformations/r/versions.png)

This feature is intended to help you **postpone** the update to a more convenient time for you in case there are
any problems with the new version. You should update the transformation code to the new version soon, as the old
version is considered unsupported.

If the `R` label is not displayed, there is no previous version offered. It is still possible to change the version
via the [API](https://developers.keboola.com/integrate/storage/api/configurations/) though.

### Memory and Processing Constraints
The Docker container running the R transformation has 16GB of allocated memory and the maximum running time is 6 hours.
The container is also limited to the **equivalent** of 2 Intel Broadwell 2.3 GHz processors.

### File Locations
The R script itself will be compiled to `/data/script.R`. To access input and output tables, use relative
(`in/tables/file.csv`, `out/tables/file.csv`), or absolute (`/data/in/tables/file.csv`, `/data/out/tables/file.csv`) paths.
To access downloaded files, use the `in/user/tag` or `/data/in/user/tag` path. If you want to dig really deep, have a look
at the full [Common Interface Specification](https://developers.keboola.com/extend/common-interface/). Temporary files can
be written to the `/tmp/` folder. Do not use the `/data/` folder for those files you do not wish to exchange with Keboola Connection.

## R Script Requirements
The R script to be run within our environment must meet the following requirements:

### Packages
The R transformation can use any package available on
[CRAN](https://cloud.r-project.org/web/packages/available_packages_by_name.html). In order for a package and
its dependencies to be automatically loaded and installed, list its name in the package section. Using `library()`
for loading is not necessary then.

The latest versions of packages are always installed. Some packages are already installed in the environment
(see [list](https://github.com/keboola/docker-custom-r/blob/master/init.R#L3)). The listed packages are installed with 
their dependencies, therefore to get an authoritative list of installed packages use the `installed.packages()` function. 
These pre-installed packages do not need to be listed in the transformation. It does no harm if you list them, but the transformation and sandbox start is slower.

In case your code relies on a specific package version, you can override the installed version by calling, e. g.:

{% highlight r %}
require(devtools)
install_version("data.table", "1.12.8")
require(data.table)
{% endhighlight %}

If you need to load a package that is on Github, but not on CRAN, then you can use the devtools::install_github method

{% highlight r %}
library(devtools)
install_github("myGithub/myRlib")
library(myRlib)
{% endhighlight %}

### CSV Format
Tables from Storage are imported to the R script from CSV files. The CSV files can be read by standard R functions.
Generally, the table can be read with default R settings. In case R gets confused, use the exact format
specification `sep=",", quote="\""`. For example:

{% highlight r %}
data <- read.csv("in/tables/in.csv", sep=",", quote="\"")
{% endhighlight %}

### Row Index in Output Tables
Do not use the row index in the output table (`row.names=FALSE`). If you are using the
[readr package](https://cran.r-project.org/web/packages/readr/readr.pdf), you can also use the `write_csv` function
which doesn't write row names.

{% highlight r %}
write.csv(data, file="out/tables/out.csv", row.names=FALSE)
{% endhighlight %}

The row index produces a new unnamed column in the CSV file which cannot be imported to [Storage](/storage/).
If the row names contain valuable data and you need to keep them, you have to convert them to a separate column first.

{% highlight r %}
df <- data.frame(first = c('a', 'b'), second = c('x', 'y'))
data <- cbind(rownames(df), df)
write.csv(data, file="/data/out/tables/out.csv", row.names=FALSE)
{% endhighlight %}

### Errors and Warnings
We have set up our environment to be a little zealous; all warnings are converted to errors and they cause the
transformation to be unsuccessful. If you have a piece of code in your transformation which may emit warnings
and you really want to ignore them, wrap the code in a `tryCatch` call:

{% highlight r %}
tryCatch(
    { ... some code ... },
    warning = function(w) {}
)
{% endhighlight %}

## Development Tutorial
We recommend that you create an [RStudio sandbox](/transformations/sandbox/#rstudio-sandbox) with the same
input mapping your transformation will use. This is the fastest way to develop your transformation code.

**Tip:** Limit the number of rows you read in from the CSV files:

{% highlight r %}
mydata <- read.csv("in/tables/mydata", nrows=500)
{% endhighlight %}

This will help you catch annoying issues without having to process all data.

You can also develop and debug R transformations on your local machine.
To do so, [install R](https://cloud.r-project.org/), preferably the same [version as us](#environment).
It is also helpful to use an IDE, such as [RStudio](https://rstudio.com/products/rstudio/#Desktop).

To simulate the input and output mapping, all you need to do is create the right directories with the right files.
The following image shows the directory structure:

{: .image-popup}
![Screenshot - Data folder structure](/transformations/r/tree.png)

The script itself is expected to be in the `data` directory; its name is arbitrary. It is possible to use relative directories,
so that you can move the script to a Keboola Connection transformation with no changes. To develop an R transformation which takes
a [sample CSV file](/transformations/r/source.csv) locally, take the following steps:

- Put the R code into a file, for instance, script.R in the working directory.
- Put all tables from the input mapping inside the `in/tables` subdirectory of the working directory.
- Place the binary files (if using any) inside the `in/user` subdirectory of the working directory, and make sure
that their name has no extension.
- Store the result CSV files inside the `out/tables` subdirectory.

Use this sample script:
{% highlight r %}
data <- read.csv(file = "in/tables/source.csv");

df <- data.frame(
  col1 = paste0(data$first, 'ping'),
  col2 = data$second * 42
)
write.csv(df, file = "out/tables/result.csv", row.names = FALSE)
{% endhighlight %}

A complete example of the above is attached below in [data.zip](/transformations/r/data.zip).
Download it and test the script in your local R installation. The `result.csv` output file will be created.
This script can be used in your transformations without any modifications.
All you need to do is

- create a table in Storage by uploading the [sample CSV file](/transformations/r/source.csv),
- create an input mapping from that table, setting its destination to `source` (as expected by the R script),
- create an output mapping from `result.csv` (produced by the R script) to a new table in your Storage,
- copy & paste the above script into the transformation code, and finally,
- save and run the transformation.

{: .image-popup}
![Screenshot - Sample Input Output Mapping](/transformations/r/sample-io.png)

### Events and Output
It is possible to output informational and debug messages from the R script simply by printing them out.
The following R script:

{% highlight r %}

print('doing something')
Sys.sleep(3)
print('doing something else')
Sys.sleep(3)
write('still doing something', stdout())
Sys.sleep(3)
write('error message', stderr())
Sys.sleep(3)
app$logInfo("information")
Sys.sleep(3)
app$logError("error")
Sys.sleep(3)
TRUE
{% endhighlight %}

produces the following events in the transformation job:

{: .image-popup}
![Screenshot - Script Events](/transformations/r/events-output.png)

The `app$logInfo` and `app$logError` functions are internally available; they can be useful if you need to know the precise
server time of when an event occurred. The standard event timestamp in job events is the time when the event was received
converted to the local time zone.

### Going Further
The above steps are usually sufficient for daily development and debugging of moderately complex R transformations, 
although they do not reproduce the transformation execution environment exactly. To create a development environment
with the exact same configuration as the transformation environment, use [our Docker image](https://developers.keboola.com/extend/docker/running/#running-transformations).

## Examples
There are more in-depth examples dealing with

- [array splitting](/transformations/r/array-splitter/),
- [plotting charts & graphs](/transformations/r/plots/), and
- [using trained models and binary files](/transformations/r/binary/).
