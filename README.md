# dbpreproc EPICS database preprocessor

dbpreproc is a powerful alternative to the [msi](http://www.slac.stanford.edu/grp/ssrl/spear/epics/extensions/msi/msi.html) tool in
EPICS. Both tools allow you to structure your EPICS databases as
'source code' and build them into one final output database when you
compile your IOC application. The IOC can then load a single output
database when it boots.

Advantages to using dbpreproc:

* Database functionality can be encapsulated in individual source
    databases, for easier comprehension.

* Generic components can be reused for shorter, simpler descriptions
  of databases (enabling easy application of [the DRY
  principle](https://en.wikipedia.org/wiki/Don't_repeat_yourself).)
  Components can be nested hierarchically.

* Embedded database parser, verifies database syntax at compile time.

* dbd verification allows every field in the database to be checked against
  the database definition at compile time. No more restarting to correct
  minor syntax errors!

* Powerful macro expansion engine - local scopes, inline default values,
  errors on missing macros.

* Automatic dependency generator for Makefile integration.

* Detailed error messages with accurate line & column numbers for
  quick and accurate debugging.


# History

dbpreproc was originally developed by Angus Gratton at Australian
National University. It is currently used on the ANU 14UD accelerator.

dbpreproc takes many ideas from [VisualDCT](http://www.slac.stanford.edu/grp/cd/soft/epics/extensions/vdct/doc/MAN-VisualDCT_Users_Manual.html#flatdb) by Cosylab, especially
VisualDCT's flatdb function.


# License

Copyright 2012 Australian National University.
Licensed under the New BSD License as described in the file LICENSE.

# Dependencies

* [pyparsing](http://pyparsing.wikispaces.com/) python parsing module, version 1.5.6 or newer
    (can be installed via *pip install pyparsing* or *easy_install install pyparsing*)


# Similarities to msi

If you're familiar with msi then most of the basic concepts in
dbpreproc will be familiar. In fact, some macro expansion features
from msi are supported verbatim:

* "substitute" and "include" directives
* Command line options to be used as an inline filter via stdin & stdout.

The msi "substitution file" format is not currently
supported. Instead, dbpreproc uses expand clauses that can be inserted
anywhere in a source database.


# Workflow

dbpreproc takes as input a "source database", which can then expand
additional source databases as included "children". It outputs a
single EPICS database that can be loaded by an IOC.

I suggest using file extension `.sdb` for source databases, but you can
use any extension you like.


# Quick Example

The source databases used in this example can be found in the dbpreproc
source under `example/`

The example assumes you have multiple vacuum gauges, model "FAKECORP
1". Each vacuum gauge is just an analog output, read by an ADC.

However, for each gauge you want the alarm status to cut in at a
different level depending on the location of the gauge.

## vacuum-fc-1.sdb

The first source database is a "child database" that generically
describes a FAKECORP 1 vacuum gauge:

    # A FAKECORP 1 model vacuum gauge

    record(ai, "$(section_name):vacuum") {
      field(DTYP, "asynInt32")
      field(INP, "@asyn($(port))")
      field(LINR, "LINEAR")
      field(EGU, "Torr")
      field(EGUL, "0")
      field(EGUF, "1")
      field(PREC, "8")
      field(HIGH, "$(warning_level|0.00001)")
      field(HSV, "MAJOR")
    }


The macro value *$(warning_level|0.00001)* in the HIGH field means
"expand the warning_level macro, or use the default value 0.00001 if
no warning_level macro is specified.")

## top-section-gauges.sdb

For the next layer up, there is a source database describing all of
the vacuum gauges that are controlled by the IOC:

    expand("vacuum-fc-1.sdb") {
      macro(section_name, "section1")
      macro(port, "adcA 0")
      macro(warning_level, "0.00001")
    }

    expand("vacuum-fc-1.sdb") {
      macro(section_name, "section2")
      macro(port, "adcA 1")
      macro(warning_level, "0.000001")
    }

    expand("vacuum-fc-1.sdb") {
      macro(section_name, "midsection")
      macro(port, "adcB 3")
      macro(warning_level, "0.0000001")
    }

Each "expand" clause specifies macros which are expanded in a new
instance of `vacuum-fc-1.sdb`.

If you left off "warning_level" from any one of the expand() clauses,
the default value would be used instead. If you left off
"section_name" from any one of the expand() clauses, dbpreproc would
report an error that the macro value was undefined when expanding that
clause.


## myioc.sdb:

The final database in the example is a "master" top-level source
database for the IOC.

This references the major functions in that IOC (in the example
there's only one major function, the vacuum gauges in the top section,
however we're assuming that the example will later be expanded to
support other, only semi-related, functions in the same IOC.)

    # IOC located near the top section of the device
    # supports vacuum monitoring, valve control, heating functions.

    expand("top-section-gauges.sdb")

## Processing the example

To process `myioc.sdb` into an output database `myioc.db`, run `dbpreproc.py`:

    dbpreproc.py -s -o myioc.db myioc.sdb

The `-s` option to dbpreproc instructs it to strip comments from the
source databases, producing a thinner output database. dbpreproc still inserts
comments describing the structure of the original source databases.

## Output database myioc.db

    # >>> expand "./top-section-gauges.sdb" at myioc.sdb:4
    # >>> expand "./vacuum-fc-1.sdb" at ./top-section-gauges.sdb:1

    record(ai, section1:vacuum) {
      field(DTYP, "asynInt32")
      field(INP, "@asyn(adcA 0)")
      field(LINR, "LINEAR")
      field(EGU, "Torr")
      field(EGUL, "0")
      field(EGUF, "1")
      field(PREC, "8")
      field(HIGH, "0.00001")
      field(HSV, "MAJOR")
    }

    # <<<< end expand "./vacuum-fc-1.sdb" at ./top-section-gauges.sdb:2
    # >>> expand "./vacuum-fc-1.sdb" at ./top-section-gauges.sdb:7

    record(ai, section2:vacuum) {
     field(DTYP, "asynInt32")
     field(INP, "@asyn(adcA 1)")
     field(LINR, "LINEAR")
     field(EGU, "Torr")
     field(EGUL, "0")
     field(EGUF, "1")
     field(PREC, "8")
     field(HIGH, "0.000001")
     field(HSV, "MAJOR")
    }

    # <<<< end expand "./vacuum-fc-1.sdb" at ./top-section-gauges.sdb:8
    # >>> expand "./vacuum-fc-1.sdb" at ./top-section-gauges.sdb:13

    record(ai, midsection:vacuum) {
     field(DTYP, "asynInt32")
     field(INP, "@asyn(adcB 3)")
     field(LINR, "LINEAR")
     field(EGU, "Torr")
     field(EGUL, "0")
     field(EGUF, "1")
     field(PREC, "8")
     field(HIGH, "0.0000001")
     field(HSV, "MAJOR")
    }

    # <<<< end expand "./vacuum-fc-1.sdb" at ./top-section-gauges.sdb:14
    # <<<< end expand "./top-section-gauges.sdb" at myioc.sdb:5

Because `myioc.db` was output by dbpreproc, you know already that it has
valid EPICS database syntax.

If you also want to confirm that all fields conform to the database
definition used by the IOC, you can run dbpreproc with the `--dbd` argument:

    dbpreproc.py --dbd /path/to/my.dbd -s -o myioc.db myioc.sdb


# Source Database Clauses

As well as plain EPICS database syntax, source databases can contain the following various clauses:

## expand

### Syntax

    expand(<sourcefile>) [ {
      macro(macroname, macrovalue)
      ...
    } ]

Recursively expands a source database as a child of the current database. Any
macros listed are defined for evaluation in the child database.

### Examples

    expand("cheese.sdb") {
      macro(name, "gorganzola")
    }

    expand("cheese.sdb") {
      macro(name, "gorgonzola")
      macro(odour_level, "9")
    }

    expand("delicious-cheeses.sdb")


Because the expanded "child" database is considered a nested scope,
any macros which are set inside that database will not be propagated
back up into the parent database.


## substitute

### Syntax

    substitute "name=value,name2=value2"

These clauses immediately substitute the given macro names for the
given macro values. The values are set in the current database, and
any child databases which are expanded are included from this one.

This is identical to the 'substitute' directive used by msi.


## include

### Syntax

    include "sourcedatabasefile"

This clause immediately includes the contents of the specified source
database. Unlike the expand clause, this is not considered a "child"
database with a separate scope - if macros are set inside the included
database, they are also set in subsequent lines of the parent
database.

This is identical to the 'include' directive used by msi.

## macro value expansion

### Syntax

    $(macro_name [ |default_value ])

Macros can be expanded anywhere that databases expect a field value,
record name or a field name. *$(macro_name)* will be replaced with the
current value of *macro_name*. Macros can be expanded recursively.

If a macro doesn't exist, dbpreproc reports an error. This behaviour
can be overriden with the `-m` command line option, which will ignore
missing macros.

Optionally, you can specify a default value for a macro by including a
pipe character (|) followed by the default value to use if the macro
is not defined. The default value itself can expand a macro.

### Simple Example

    record(ao, $(name)) {
     field(DTYP, "asynInt32")
     field(DESC, "$(desc)")
     field(OUT, "$(port)")
     field(LINR, "LINEAR")
     field(EGUF, "$(eguf|10)")
     field(EGUL, $(egul|0)")
    }

This record can be expanded minimally:

    expand("mydac.sdb") {
      macro(name, "dac1")
      macro(port, "@asyn(dacdac 0)")
      macro(desc, "My favourite DAC")
    }

Or alternatviely, you can set macros named eguf & egul to override the
default values of 0-10.

    expand("mydac.sdb") {
      macro(name, "dac2")
      macro(port, "@asyn(dacdac 1)")
      macro(desc, "My 5v bipolar DAC")
      macro(eguf, "5")
      macro(eful, "-5")
    }

### Powerful Example

This example shows some more powerful capabilities of combining macro expansion features.

    # analog readback
    #
    # use 'name' macro to set name
    # Optionally set macro 'prec' to precision, default is 3.
    #
    # Operator range:
    # Can be set in any of 3 different ways:
    # * Set macros 'low' and 'high' and the range becomes $(low) to $(high).
    # * Set the macro 'limit' and the range becomes -$(limit) to +$(limit).
    # * Set only the macro 'high' and the range becomes 0 - $(high)
    record(ai, $(name)) {
      field(PREC, "$(prec|3)")
      field(LOPR, "$(low|-$(limit|0))")
      field(HOPR, "$(high|$(limit))")
    }


# Integration with EPICS Build System

dbpreproc can be easily integrated into an existing EPICS App database build system. Add rules like this to your TOP/configure/RULES file::

    $(COMMON_DIR)/%.db: $(COMMON_DIR)/../%.sdb $(INSTALL_DBD)
    	dbpreproc.py --dbd $(INSTALL_DBD)/mydbdfile.dbd --dbd-cache dbd.cache -MF $(@:.db=.d) -s -I $(COMMON_DIR)/.. -o $@ $<

    include $(wildcard $(COMMON_DIR)/*.d)

This rule assumes that for any output `mydb.db` file, there is a source file `mydb.sdb`. For example, if you create `myApp/Db/mydb.sdb` you also insert into `myApp/Db/Makefile`:

    DB += mydb.db

So that `mydb.sdb` gets processed to create output database `mydb.db`

## PATH

This example also assumes dbpreproc.py is on the PATH. Otherwise you
can specify a DBPREPROC variable with the full path, and change the
second line to start with *$(DBPREPROC)/dbpreproc.py*

## Command Line Options

The additional options given in the example RULES are as follows:

**--dbd $(INSTALL_DBD)/mydbdfile.dbd --dbd-cache dbd.cache**

You'll need to change "mydbdfile.dbd" to be the name of your dbd file.

This causes database output to be automatically verified against the
dbd file. The `--dbd-cache` option speeds up generation by only parsing
the dbd file when it changes. The cache file is created in the
`O.Common` directory and automatically removed during 'make clean'.

**-MF $(@:.db=.d)**
combines with
**include $(wildcard $(COMMON_DIR)/*.d)**

The `-MF` option produces a Make-compatible mydb.d file
listing the source database files that are dependencies for the output
database. This means the output database will be automatically
regenerated by make if any of the source files change, but not otherwise.

**-s**

This option strips comments from the input databases before writing
them to the output database.


# Development & Bugs

If you find bugs in dbpreproc, please report them via the [Issues page](https://github.com/anunuclear/dbpreproc/issues) on
Github. Please include a sample if the bug is a parsing problem with a
particular snippet of database or database definition format text.

Patches, pull requests and other contributions are always welcome as well!

The dbpreproc parser has a unit test suite which ran be run from the dbpreproc
directory via **./tests.py -b**. If adding parser features or fixing
complex bugs, I very much recommend test driven development - write a
test database for the feature/bug first under the `testdata/` directory,
then write unit tests that fail. Then bugfix until all tests pass. :)


