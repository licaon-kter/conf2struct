SYNOPSIS
========

 conf2struct <file.cfg>

DESCRIPTION
===========

`conf2struct` takes a configuration file that describes a
configuration file in the [libconfig](https://hyperrealm.github.io/libconfig/) format, and generates a C parser that will read a configuration file directly into a C structure. The goal is to accept any file that is valid for `libconfig`, which in particular means using `conf2struct` does not introduce restrictions to what the configuration file should look like.

`conf2struct` manageѕ optional settings and default values.

(A bit of history:
[sslh](http://www.rutschle.net/tech/sslh/README.html) uses
`libconfig`, which provides a C API. There are two downsides
to that: with the increase of the number of settings, the
code to read the configuration file grew and became harder
to read, while being all mostly boilerplate code; also,
every time it reads a setting `libconfig` traverses a tree
doing string compares on the setting names, which is very
inefficient for settings that get used very often: those
setting need to be copied to a structure. Put both problems
together, and you get this program as a solution by taking a
step up.)


Configuration file
==================

The format of the target configuration file is itself
described as a configuration file which is somewhat similar
to a schema file.

Global declarations
-------------------

The following settings are located at the root of the
configuration file:

- `header`: output file name for the struct declarations 
- `parser`: output file for the C code
- `config`: description of the target configuration file.
This contains a `name` entry which will prefix all
symbols, and a `items` declaration equivalent to that of a
group (see below).

Settings
--------

The following entries are mandatory for each setting:

- `name` specifies the name of the setting
- `type` specifies the type of the setting. It is one of the
  libconfig types: `boolean`, `int`, `int64`, `float`,
`group`, `list`, `array`. Type can also be specified as `runtime` if
you want to add runtime data to the struct. The C type will
then be specified by the additionnal `c_type` setting. `c2s`
will do nothing with these types (no init, no alloc, no printing).


Scalars
-------

Scalar types take the additional following entries: 

- `default` specifies its default value. If the setting has no default,
then it is mandatory.
- `optional` specifies that a setting can be undefined.
  Undefined strings will point to NULL, and optional
settings receive an additional `<id>_is_present` boolean in the
final struct.

Groups
------

Group entries must contain an `items` setting which contains
a list of named settings. Groups can contain items of any type.

Lists
-----

List entries must contain an `items` setting which contains
the list of setting expected in each item of the list.
Lists can contain anything, including other lists.

Note this does not support items of varying types, as is
possible in libconfig. This libconfig feature feels
counter-intuitive when targeting a struct for storage.

Arrays
------

Settings of type `array` must contain an `item_type`
setting which specifies the type of the scalars contained in
the array.

Lists and arrays both get converted to a static array. A
field `<name>_len` is added containing the number of
elements in the array.


Parser API
==========

Assuming the prefix for our configuration is `foo`, the prototype for the parser produced is:

```
int foo_parse_file(
        const char* filename,
        struct foo_items* config, 
        const char** errmsg);
```

- `filename` is the name of the file, passed directly to
libconfig.

- `config` is the root configuration struct as output by
  `conf2struct`. The parser will fill that structure and
allocate memory for groups, lists and arrays as required.

- `errmsg` is a text string that explains what went wrong if
  parsing failed (e.g. a mandatory option is missing).

`config_parser()` returns 0 on failure, 1 on success.

Printer API
===========

`conf2struct` also builds a pretty-printer that takes a
`struct` as input.

Example
=======

The file `eg_conf.cfg` documents the `conf2struct`
configuration to build a parser for the libconfig
`example.cfg`. `parser.c` shows a very simple parser that
also reports errors, using this example. A simple:

```
make
./example
```

will produce the parser in `example.c` and `example.h` (as
specified in `eg_conf.cfg`), which parses `example.cfg` and
prints the result directly from the in-memory struct.

confcheck
=========

`conf2struct` is written in Perl, and the Perl
implementation for `libconfig` appears to be very bad at
reporting parsing errors (that, or the documentation is very
                          bad). The small `confcheck`
program can be used to validate configuration files: it will
provide the exact parse error location, if any.


TODO
====

- validation parameters, e.g. `min_length` / `max_length` for arrays and
lists, and strings, which can be verified to check the validity
of the configuration.

- generator of a command-line parser to override the
configuration file.

Similar project
===============

[x2struct](https://github.com/xyz347/x2struct) does a
similar job, but for C++, added constraints to the target
configuration file (while I did not want users to have to
change existing configuration files), and I didn't see how
to extend it to get useful error messages, default values,
and setting validation.

