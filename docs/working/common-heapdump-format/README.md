# Extensible heap dumps for Node programs

## Status

**This is still very much a draft document.**


## Motivation

Postmortem debugging is used to understand the behavior of programs (usually
broken programs).<sup>3</sup>  Problems that people solve with postmortem
debugging include:

* fatal failure: crashes (throwing an uncaught exception, or any other kind of
  programmer error<sup>4</sup>)
* explicit, non-fatal failure: programs emitting error messages or responding to
  requests with explicit failures
* implicit, non-fatal failure: memory leaks or other excessive memory usage;
  incorrect behavior (e.g., incorrect output)

Several tools exist to allow engineers to interactively explore the heap of
Node.js programs<sup>1,2</sup>.  These tools, which operate on both live
processes and OS core dumps, consist of two parts: interpreting the memory
associated with the Node program to construct an internal model of the program's
JavaScript-level state, and then analyzing and presenting that model in a way
that's useful for end users.

People have expressed interest in decoupling these steps so that users could use
one tool to read the memory dump and produce a file describing all of the
JavaScript-level state in the target and then feed this into any number of
analysis and visualization tools to help understand that dump.  There are
several reasons this would be useful:

* The existing tools for interpreting memory dumps tend to be somewhat
  platform-specific or build-specific because they rely on system facilities for
  reading memory dumps or build-time facilities in Node for making sense of
  those dumps.  Separating the analysis and visualization would allow people
  using different toolchains to collaborate on the analysis and visualization
  part of the problem.  For example, we could build a heap visualizer using the
  common format, which everyone could use regardless of their choice of tool for
  generating the common format in the first place.
* The skill sets required to solve the two parts of this problem are pretty
  different, and there are different developers interested in working on them.
* Often, the process of scanning memory snapshots is very time-consuming, and
  the results today exist only inside the debugger.  It would be useful if these
  results could be persisted so that users could save time in future debugging
  sessions.

This document proposes a file format for the exchange of JavaScript-level state
snapshots constructed by postmortem debugging tools.  The goal is that this
format could be exported by today's existing Node.js postmortem debugging tools
and could be imported both by those same tools (to resume previous debugging
sessions) and by other analysis and visualization tools.

This document is based loosely on a prototype implemented for
mdb\_v8<sup>5</sup>, except that the physical representation suggested here is
pretty different.


## Design goals

* Given the goal of debugging all kinds of software failure, the format needs to
  be able to express any JavaScript-level state contained in the Node program.
  That includes all JavaScript heap objects, including primitive types, as well
  as stack traces of JavaScript threads (including function arguments) at the
  point where the snapshot was generated.
* Ideally, the file format should be extensible so that debuggers can include
  whatever extra data they see fit, like information about the source of the
  data (process identifier or core file), the software that generated the dump,
  and the like.
* Even the descriptions of JavaScript values should be extensible.  Originally,
  postmortem tools only supported operations like printing out JavaScript
  objects, but more recent versions added support for printing out closure
  variables.  It's important that it be possible for the format to incorporate
  additional information like this as debuggers gain deeper introspection
  abilities.  These extended properties should be namespaced to avoid conflicts.
* The representation of JavaScript values should reflect the well-established
  internal V8 structures (e.g., SMIs vs. other heap values).  This unfortunately
  couples the format to V8's implementation, but we believe this is necessary to
  avoid leaky abstractions and to obtain an accurate view of memory usage and
  the relationships between objects.
* The format should be easy to work with from a variety of programming
  environments on a variety of platforms.

It's an open question whether the file format should be self-indexing.  See
"Physical format" below.


## Prior work

Core files themselves represent a form for storing all this data (and much
more), but for the reasons described above, it's preferable to have a format
specifically organized for conveying JavaScript state.

Java has at least two heap dump formats: a classic text-based format<sup>8</sup>
and a portable binary format<sup>9</sup>.

The only format that we're familiar with that serializes a Node program's state
is the V8 heap dump format.  It does not appear to have a formal description,
but there's some documentation in source code<sup>6</sup> and other web
sites<sup>7</sup>.  It has several limitations:

* It does not include primitive types other than strings, including small
  integers, booleans, `undefined`, and `null`.  (The main design goal appears to
  be to understand memory usage rather than other kinds of brokenness.)
* It does not contain information about a current call stack, although it does
  have room for profiled call stacks.
* The use of monotonic ids assigned to nodes and edges makes it difficult to
  generate without additional O(N) memory usage.
* Relatedly, the single large JSON object makes it hard to parse without using
  O(N) memory.  Streaming JSON parsers could alleviate this, but do not appear
  to be widely used in Node.js.
* The heap dump does not preserve any connection between nodes and their
  addresses in memory, which makes it hard to build a debugger for both
  JavaScript and native code (e.g., to present information about the libuv
  structure and file descriptor associated with a Node `Stream`).

There are a few major advantages of the V8 heap dump format:

* Most importantly, there are many existing tools that generate and consume it.
  An important design goal for this format is that it should be possible to
  translate it easily to or from a V8 heap dump, even if some information is
  lost in the process.
* The format itself is somewhat extensible: structures used in the file are
  described in the file itself, so it's possible for implementors to add
  additional fields.

The format described here borrows a number of ideas from the V8 heap dump
format, but in a package that we hope will be easier to generate and process,
and with the ability to incorporate more kinds of information.


## Logical format

We will first explain the logical contents of the file in terms of the records,
types, fields, and what they contain.  Under "Physical format" below, we'll
discuss possible ways to represent these in a file.


### Extensible format: records, types, subtypes

To accomplish all this, the file format logically consists of **records**, each
of which has one of a small number of **types**.  These types are defined within
the file itself.  Built-in top-level types include:

* **metadata**: key-value pairs (with strings as both keys and values) are used
  for debuggers to incorporate their own metadata.  This is intended for small
  amounts of non-program data (like the generating program's name and version).
  Other types of program state should be represented with other types of
  records.
* **nodes**: represent JavaScript values allocated on the heap
* **edges**: represent relationships between nodes.
* **strings**: represent the contents of JavaScript strings as well as symbols
  used in the program

The `type` of a record defines what properties it has.  For example, `metadata`
records have fields `key` and `value`.  This is recursive: `node` and `edge`
records have a subtype field that determines what other properties they have.
For example, a `date` node has an underlying `value` field representing
milliseconds since the Unix epoch.  A `regular expression` node has underlying
fields representing the `source` string and `flags` string of the regular
expression.

There are no strings contained within `node` or `edge` records.  A JavaScript
string is represented with a node, but it refers to a separate `string` record
that contains the data for that string.  This allows `node` records to have
bounded, well-known sizes.  Similarly, contents of strings used as object
property names or variable names appear in `string` records.

**The types and fields available are defined within the file itself.  This
document describes suggested fields and types by name, but debuggers may omit
types and fields (because they do not support them) or include additional
fields.**  For example, a debugger that understood Node timeout objects could
include another top-level type for the timeouts that are registered in the Node
program.  When including additional fields, these should be namespaced to avoid
collisions with other implementations.

The numeric values for these types and subtypes are not specified here, and
should be read from the file itself.


### Summary of Nodes and their subtypes

#### Node identifiers and JavaScript values

As mentioned above, nodes represent heap-allocated JavaScript values.  All nodes
have an **identifier**, which is a 64-bit numeric value.  This number can be
any unique value, and is likely either from a set of increasing integers
(similar to the V8 heap dump format) or (more usefully) the hexadecimal-encoded
address where the object exists in the memory snapshot.  JavaScript programs
processing these dumps must be careful when working with 64-bit integer values,
since these are not natively supported by JavaScript.

Small integers are not represented with nodes.  V8 represents small integers
inline where they're used, rather than allocating them separately on the heap.
Any field in this file that represents a JavaScript value may be _either_ a Node
identifier (described above) or a small integer.  This format uses the same
convention that V8 uses internally: numeric values with the lowest bit set
denote Node identifiers.  Numeric values without that bit set represent small
integers, and their actual value is obtained by shifting the numeric value right
by one bit.


#### Node subtypes

Nodes have several subtypes:

* **array**: a JavaScript array
* **object**: a JavaScript object, not including `null`
* **flat string**, **concatenated string**, or **sliced string**: these
  represent various kinds of JavaScript strings.
* **code**: represents a chunk of executable machine code, which may be part of
  a function or other chunks of VM-generated code
* **closure**: represents a JavaScript function and the context in which it was
  created.
* **function metadata**: represents a single logical function as programmers
  tend to think of them, being defined at a single location in the source code.
  For example, you may have a single named function called `foo()`, but there
  may be a large number of _closures_ for that function.  These closures are
  sometimes referred to as functions, which makes the terminology a little
  confusing.)
* **regular expression**: represents an instance of the JavaScript RegExp class.
* **date**: represents an instance of the JavaScript Date class.
* **heap number**: represents a number allocated on the heap
* **native**: represents a chunk of memory outside the JavaScript heap
* **oddball**: represents a boolean, `null`, `undefined`, or other special
  value

The extra fields associated with these subtypes are described below.


#### Strings

Strings are worth paying particular attention to.  V8 represents strings in a
few different ways, and these impact their memory usage and the way other
objects refer to them, so these representations are preserved in this format.
The actual string subtypes are:

* **flat string**: a JavaScript string represented by a sequence of characters
* **concatenated string**: a JavaScript string represented by a concatenation of
  a flag string, another concatenated string, or a sliced string
* **sliced string**: a JavaScript string represented as a slice of a flat
  string, concatenated string, or another sliced string.


### Edge subtypes

Edges have a few subtypes:

* **object property**: an edge from node `O` to value `V` with label `K` means
  that object `O` has a property called `K` with value `V`.  Note that `K` will
  have a symbol record in the file.
* **array element**: an edge from node `A` to value `V` with label `i` means
  that array `A` has a property called `i` with value `V`.  Note that `i` is a
  string (per the JavaScript standard) and will have a symbol record in the
  file.
* **closure variable**: an edge from node `C` to value `V` with label `N` means
  that function closure `C` has a reference to value `V` using a closure
  variable called `N`.  `N` will have a symbol record in the file.

In all of these cases, the values `V` are JavaScript values, as described under
"Node identifiers and JavaScript values" above.

## Record format reference

This section describes the fields associated with each kind of record.

### Metadata

Metadata records have two fields:

Field name | Type          | Meaning
---------- | ------------- | -------
key        | inline string | identifies this piece of metadata
value      | inline string | varies based on key

Interpretation of these records is up to the generating and consuming programs
except as recommended below.  Multiple records with the same `key` are allowed,
with semantics varying by key.

Suggested metadata fields include the following:

Metadata key        | Meaning of value
------------------- | ----------------
`version_major`     | major version of this file format being used
`generator`         | name of the program that generated this file
`crtime`            | ISO 8601 timestamp when this file was generated
`target_pid`        | process identifier for the process from which this dump was generated
`target_file`       | name of the core file from which this file was generated (if any)
`target_hostname`   | hostname of the system where the target process was running
`target_source`     | one of: "core", "process", or another value describing how this file was generated
`target_signal_sig` | if the process was terminated by a signal, the signal name (e.g., "SIGINT", not "2")
`target_signal_pid` | if the process was terminated by a signal, the sending pid
`target_psargs`     | human-readable summary of the process arguments
`target_model`      | target process's data model, either "ILP32" or "LP64"

The only metadata record that's required is for key `version_major`, which for
this version of the file format should have value `"1"`.


### Nodes

All nodes have the following fields:

Field name   | Type                  | Meaning
-----------  | --------------------- | -------
`identifier` | 64-byte numeric value | See "Node identifiers and JavaScript values" above.
`subtype`    | Number                | Identifies the type of object represented, which identifies which other fields are present

Subtype **array** has the field:

Field name | Type           | Meaning
---------- | -------------- | -------
`length`   | 32-bit integer | Matches the `length` JavaScript property for this array.

Subtype **object** has the field:

Field name    | Type  | Meaning
------------- | ----- | -------
`constructor` | value | the Closure used to construct this object

Subtype **flat string** has fields:

Field name    | Type  | Meaning
------------- | ----- | -------
`length`      | value | the number of characters in the string (may be a HeapNumber)
`data`        | value | native address where the characters live

Subtype **concatenated string** has fields:

Field name    | Type  | Meaning
------------- | ----- | -------
`length`      | value | same as for flat string
`s1`          | value | the first half of the string
`s2`          | value | the second half of the string

The contents of the string is constructed by concatenating `s1` with `s2`.  This
process is recursive: Either or both of `s1` or `s2` may be any kind of string.

Subtype **sliced string** has fields:

Field name    | Type  | Meaning
------------- | ----- | -------
`length`      | value | same as for flat string
`source`      | value | source string (see below)
`offset`      | value | offset into `source` where this string starts

Subtype **code** has no fields at this time.

Subtype **closure** has fields:

Field name    | Type  | Meaning
------------- | ----- | -------
`metadata`    | value | reference to function metadata (see below)

Subtype **function metadata** has fields:

Field name    | Type  | Meaning
------------- | ----- | -------
`name`        | value | the string name assigned to the function when it was defined
`script_name` | value | the string name of the script where the function was defined
`position`    | value | the numeric offset into the script where the function was defined

Subtype **regular expression** has fields:

Field name   | Type  | Meaning
------------ | ----- | -------
`source`     | value | string representing the regular expression itself
`flags`      | value | string representing the enabled regexp flags

Subtype **date** has fields:

Field name   | Type  | Meaning
------------ | ----- | -------
`timestamp`  | value | number denoting milliseconds since the Unix epoch.


Subtype **heap number** has fields:

Field name   | Type                 | Meaning
------------ | -------------------- | -------
`value`      | floating-point value | floating-point representation of a numeric value

Subtype **native** has fields:

Field name   | Type                 | Meaning
------------ | -------------------- | -------
`address`    | 64-bit numeric value | address of native chunk of memory

Subtype **oddball** has fields:

Field name | Type  | Meaning
---------- | ----- | -------
`name`     | value | the string name of the oddball, which is generally one of: `null`, `true`, `false`, `undefined`, or `the_hole`.


### Edges

All edges have the following fields:

Field name | Type   | Meaning
---------- | ------ | -------
`subtype`  | Number | Identifies the type of edge represented, which determines what it represents
`source`   | value  | See below.
`dest`     | value  | See below.
`label`    | value  | String (a symbol).  See below.

The subtypes are:

Subtype              | Details
-------------------- | -------
`"object property"`  | `source` is an object with a property named `label` and value `dest`.
`"array element"`    | `source` is an array with an element named `label` and value `dest`.
`"closure variable"` | `source` is a closure with access to a variable called `label` that has value `dest`.

As mentioned above, values may be either node identifiers or small integers.
Note that per JavaScript, array element names are technically strings, not
integers.


### Strings

String records represent string _contents_, not JavaScript strings _per se_.
(Those are represented with nodes that refer to these string contents.)  String
data are identified by 64-bit numeric string identifiers (similar to node
identifiers) and contain a chunk of UTF-8-encoded characters.


## Physical format

The above describes the kinds of records contained in the file, but not how they
are encoded into the file.  There are two obvious ways of encoding records
records into the file:

* as a flat file with a list of records, in any commonly-parseable format:
  newline-separated JSON, BSON, or the like.  In this implementation, we would
  say that the records could appear in any order, and it's up to the consumer to
  make sense of that.  (This is because with a graph structure, it's impossible
  to avoid including references to objects not yet defined, so consumers have to
  be able to keep some sense to make sense of the file anyway.)
* as a more complex file with its own index, which would allow for random-access
  and complex queries without having to read the entire file or store the
  contents in memory

The flat file is easier to export and import directly between memory and disk.
However, for large dumps, it could take a long time to import, defeating one of
the design goals.  Besides that, even trivial usage of it would likely require
reading the whole file and likely storing it in memory.

A sqlite database file would be a convenient example of the file containing its
own index.  In this case, the heap is represented as a denormalized SQL
database.  sqlite is easy to embed, and bindings exist for many environments.
The heap information could be queried directly without having to read the whole
file.

Both approaches are equally expressive, and it would be possible to write a
translator from either format to the other.

### Flat file approach

With any of the flat file approaches, the suggested layout is a sequence of
records:

* Metadata records should come first (but don't have to, so that it's possible
  to include metadata later in the file).
* Types and subtypes must appear before any "node" or "edge" records.  We would
  need to figure out how to best express these, and there are existing
  conventions for doing this.
* Nodes and edges may appear in any order at all.


### Database approach

Using the sqlite representation, the database schema consists of tables
corresponding to the values described above.  Specifically:

The **metadata** table has columns `key` and `value` as described in the logical
reference above.

The **node** table has columns `identifier` and `nodetypeid` as described in the
logical reference above

The **node_types** table has columns `nodetypeid` (a number), `name` (a string), and
`table_name` (a string).

The **array**, **object**, **string_flat**, **string_cons**, **string_sliced**,
**closure**, **function_metadata**, **regular_expression**, **date**,
**heap_number**, **native**, and **oddball** have the same columns described in
the logical reference above, plus a `node_identifier` column.

The **edge** table has columns `edgetypeid` (a number), `source`, `dest`, and
`label` as described in the logical reference above.

The **edge_types** type has columns `edgetypeid` (a number) and `name` (a
string).

The **strings** table has columns `stringid` (a number) and `data` (a
UTF-8-encoded string).


## TODO

* Decide on which physical format path to take
* Add examples
* Prototype importer and exporter


## References

1. [Core dump debugging for the IBM SDK for
   Node.js](http://www.ibm.com/developerworks/library/wa-ibm-node-enterprise-dump-debug-sdk-nodejs-trs/index.html)
2. [mdb_v8: postmortem debugging for Node.js](https://github.com/joyent/mdb_v8)
3. [Postmortem Debugging in Dynamic
   Environments](https://queue.acm.org/detail.cfm?id=2039361)
4. [Error Handling in
   Node.js](https://www.joyent.com/developers/node/design/errors)
5. Prototype [heap dump implemented for
   mdb_v8](https://github.com/joyent/mdb_v8/blob/fd353b22035be8e4f0d4bd98e6df143184923866/src/mdb_v8.c#L5224-L5299)
6. Description of [V8 heap dump format](https://github.com/v8/v8/blob/master/include/v8-profiler.h) in v8 source code
7. Summary of [terms used in V8 heap
   snapshots](http://bmeck.github.io/snapshot-utils/doc/manual/terms.html)
8. [Text (classic) Heapdump file
   format](http://www-01.ibm.com/support/knowledgecenter/SSYKE2_8.0.0/com.ibm.java.lnx.80.doc/diag/tools/heapdump_classic_file_format.html?lang=en) for Java programs
9. [Portable Heap Dump (PHD) file
   format](http://www-01.ibm.com/support/knowledgecenter/SSYKE2_8.0.0/com.ibm.java.lnx.80.doc/diag/tools/heapdump_phd_format.html?lang=en) for Java programs
