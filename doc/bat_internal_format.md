This document describes the format of the datastructures used internally by BAT.

For each file that is unpacked and scanned by BAT data is kept, including information about the path, unpacked files (children), tags, and so on.

Because the data can grow quite big (especially when dealing with packages with possibly millons of matches, like the Linux kernel) the data in BAT is kept in two separate data structures to keep memory usage (fairly) low. One part is kept in memory, the other part is kept on disk and (de)serialized on a per need basis:

* unpackreports: this is a list of dictionairies containing metainformation about every file. This list is always kept in memory during the entire scan. Information stored includes the path, checksums, tags, offsets and sizes of filesystems/compressed files contained in the file (children), names of files that were unpacked from this file, ranges of 'blacklisted' bytes that don't need to be (re)scanned, and so on. The key of each dictionary is the path of a file that is unpacked by BAT. Files that are duplicates (same checksum) are tagged as such. This list is written to a Python pickle called "scandata.pickle" or a JSON file called "scandata.json" (or both, depending on the configuration) at the end of the scan.

* leafreports: for each file (except empty files, symbolic links, pipes, sockets, etc.) a data structure with all the results of the specific scans is kept, such as matching data, but also different data. Because of memory constraints this data is kept on disk, not in memory, in a Python pickle file for each checksum (unique file). The data is read from disk on a per need basis. The Python pickles (optionally gzip compressed) can be found in the directory "filereports" inside the scan result archive.

Some data is duplicated between the two:

* tags: tags are used by many scans to determine whether or not the scan should be run for the file. To prevent that each scan reads the pickles with the leaf reports the data in tags of the leaf reports is duplicated in the entries for files in unpackreports. In the unpackreports there might be additional tags, such as "duplicate" which are not present in the leaf reports.

Data kept in unpackreports

The following data is packed in each entry for unpackreports:

* name: name of the file (without directory component)
* path: relative directory path inside the unpacked file system or compressed file
* realpath: full absolute directory used during the scan
* relativename: relative path inside the unpacked file system or compressed file (effectively combining realpath and name, but relative to the scanning directory)
* magic: output of libmagic. This is dependent on the version of libmagic and the configuration of the host system and is only used for informational purposes, not for any determination, as libmagic only looks at a limited portion of the file.
* tags: list of tags as given to the file by BAT
* size: size of the file in bytes. This field only exists for real files, not for directories, pipes, symbolic links, fifos, device files, and so on.
* checksum: checksum of the file. This only exists for real files, not for directories, pipes, symbolic links, fifos, device files, and so on.
* sha256/sha1/md5/tlsh: various hashes for files
* scans: List of scan results if a file system or compressed file could be found inside. This only exists for real files, not for directories, pipes, symbolic links, fifos, device files, and so on. Each scan result is a dictionary with results.

The top level file (the root) has an additional item:

* scandate: date + time the scan was started

Data kept in "scans" (unpackreports)

If any data could be unpacked/carved from a larger file the 'scans' element for a file in unpackreports will be a list with results. Each result is a dictionary with the following elements:

* offset: data offset in the original file where the file system/compressed file starts
* size: size of the file system or compressed file in the original file. This element will sometimes be missing.
* scanreports: list of files that were unpacked from the file. These correspond to other elements in unpackreports
* scanname: the name of the scan that was used to unpack/decompress the file, for example 'gzip' or 'lzma'

Data kept in leaf reports

The leaf reports are kept in (optionally gzip compressed) Python pickle files. These can be found in the directory "filereports" in the scan result archive generated by BAT.

The data is stored as a dictionary inside each leafreport. The keys for this dictionary are either the values of the "name" configuration of every scan in the configuration file, or if these are not present the names of the scans from the configuration file that was used by BAT during scanning. Also included is the key "tags". There is no standardized output format for the leaf scans, but it is up to the leaf scans and code that processes the results.

* tags: a list of tags

What follows is an explanation of the data structure returned by each of the currently supported leaf scans.

Data for the architecture scan

* architecture: line of CPU architecture as reported in the "Machine" field in the output of "readelf -h", for example "MIPS R3000"

Data for the kernelmoduleversion scan

Data for the identifier scan

The identifier scan extracts identifiers from a binary. These are later processed by for example the ranking scan. The data is stored as a dictionary (the datastructure has changed compared to BAT 19 and earlier versions).

The first element is a list of string identifiers as extracted by running "strings" on the binary. The second element of the tuple is a dictionary that contains various other symbols than string identifiers. Possible keys of the dictionary are:

* strings: list of string identifiers
* functionnames: a set of function names
* variablenames: a set of variable names
* kernelsymbols: a list of Linux kernel symbols
* kernelfunctions: a list of identifiers extracted and identified as Linux kernel functions
* language: language family that BAT thinks the source code files (that were used to generate the binary) are part of.

Data for the ranking scan

The current ranking scan stores its results as a tuple with the following elements:

* string identifier results: a dictionary with various results for matching string identifiers
* function name results: a dictionary with various results for matching function names
* variable name results: a dictionary with various results for matching variable names
* language: language family that BAT thinks the source code files (that were used to generate the binary) are part of.

Please note that this format will likely change in the future.

Data for string identifier results (ranking scan):

* extractedlines: number indicating how many lines were extracted from the binary
* matchedlines: number indicating how many lines were matched to the database
* unmatchedlines: number indicating the number of unmatched lines
* unmatched: list of unmatched lines
* matcheddirectassignedlines: number indicating number of matches that could directly be assigned to a package
* matchednonassignedlines: number indicating number of matches that could not be assigned (score too low)
* matchednotclonelines: number indicating number of matches that were matched, but which are not cloned between packages (needs work)
* reports: a list with the results (rank, unique matches, licenses, etc.) per package
* nonUniqueMatches: dictionary listing per package the non unique assignments (package name as key, list of lines as value)
* nonUniqueAssignments: dictionary listing non unique assignments per package (package name as key, number of lines assigned as value)
* scores: dictionary listing computed score per package (package name as key, score as value)

Data stored in 'reports' (string identifier results)

The result stored in 'reports' is an 8 tuple with the following information:

1. rank: number indicating the rank (numbered from 1 for the highest ranked package)
2. package name: name of the package
3. unique: unique matches for a package
4. uniquematcheslen: amount of unique matches
5. percentage: percentage of the score computed by ranking scan
6. packageversions: dictionary listing per version of a package how many strings were matched
7. packagelicenses: list of tuples (license, scanner) of licenses that were found. These results are not per result, but per package.
8. packagecopyrights: currently unused, but will be used for storing copyrights in the same way as licenses

Data stored in unique matches (string identifier results)

The unique matches data stored in 'reports' is a list of tuples. The tuple has two elements:

* string identifier: the actual matched string
* a list of tuples with match information. Each tuple consists of:
  * a checksum (SHA256)
  * line number of the match
  * a list of tuples (package version, file inside the package)

Data for function name results (ranking scan):

The function name results dictionary has the following keys. Sometimes these keys are missing (meaning: no results).

* uniquepackages: dictionary, packages as keys, values are list of function names unique for that package. This data can also be retrieved by analysing "versionresults"
* totalnames: number of function names that was extracted from the dynamic linking section of the binary
* namesmatched: number of function names that were matched with function names in the database
* uniquematches: number of unique matches, can also be retrieved by analysing either "uniquepackages" or "versionresults"
* versionresults: dictionary, packages as keys, values are lists of tuples with results. This data can also be retrieved by analysing "versionresults".
* packages: dictionary, packages as keys, values are a list of tuples (version, number of unique results)

The matches data stored in 'versionresults' is a list of tuples. The tuple has two elements:

* function name identifier: the actual matched function name
* a list of tuples with match information. Each tuple consists of:
  * a checksum (SHA256)
  * line number of the match
  * a list of tuples (package version, file inside the package)

Data for version name results (ranking scan):

The variable name results dictionary has the following keys:

* uniquepackages: dictionary, packages as keys, values are list of varibale names unique for that package. This data can also be retrieved by analysing "versionresults"
* totalnames: number of function names that was extracted from the dynamic linking section of the binary
* allvariables: dictionary, variable names as keys, list of packages (no version number) in which they occur as values.
* versionresults: dictionary, packages as keys, values are lists of tuples with results. This data can also be retrieved by analysing "versionresults".
* packages: dictionary, packages as keys, values are a list of tuples (version, number of unique results)

The matches data stored in 'versionresults' is a list of tuples. The tuple has two elements:

* variable name identifier: the actual matched variable name
* a list of tuples with match information. Each tuple consists of:
  * a checksum (SHA256)
  * line number of the match
  * a list of tuples (package version, file inside the package)

Data used for generating matching piecharts

Two types of piecharts are generated: piecharts depicting how the strings were matched and a piechart for the score. For the piechart displaying statistics about how strings were matched (stored as $checksum-statpiechart.png) the following information from the ranking scan and 'reports' is used:

* unmatchedlines (from ranking scan)
* matchednonassignedlines (from ranking scan)
* matchednotclonedlines (from ranking scan)
* nonUniqueAssignments (from ranking scan, will likely be replaced by nonUniqueMatches)
* package (from 'reports')
* unique (from 'reports')

These are used to generate a list of labels and a list of numbers which are then used to create a slice of the piechart and label it.

For the piechart for the score the following data from 'reports' is used:

* percentage
* package name

Each combination (package name, percentage) is then used to create a slice of the piechart and label it.

Data used for generating matching barcharts

Data generated by security scanning

There are various security related scans in BAT:

* virus: a test to check for the presence of viruses, based on ClamAV
* shellinvocations: a test to check for possible attack vectors using shell invocations in binaries

Data generated by virus scan

The virus scan returns the virus name as determined by ClamAV, if any virus is found.

Data generated by shell invocations scan

The shell invocations scan outputs a Python set with strings that contain a possibly buggy shell invocation.