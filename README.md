# Include dependendcy viewer

This program generates [dot](https://graphviz.org/doc/info/lang.html) graph
descriptions for include headers for c/c++.

The intent is to visualize dependencies among files.

## Functionality

- select what files to parse
- define how to group to viewed files
- abiltiy to [compile database](https://clang.llvm.org/docs/JSONCompilationDatabase.html) 
  files to figure out imports. These files can be generated by gn/cmake/other tools.
- abiltiy to use [gn](https://gn.googlesource.com/gn/) to auto-group headers

## Examples

Here is an example complex include graph:

<img src="examples/example_complex_includes.png" alt="Include dependencies" width=800 />

Here is an example of how a complex zoomed-in dependency group looks like:

<img src="examples/example_zoom.png" alt="Include dependencies" width=800 />

## Building

This is a standard rust binary:

- `cargo build --release` will generate an output in `target/release/include-graph`
- `cargo run -- <args-here>` can be used to run in-place (and debug)
- `cargo test` or `cargo nextest run` will execute some the integrated unit tests (increasing
  test coverage is still a work in progress)

## Usage

```sh
# Simple run using a configuration database
# if no `output` is provided, the output will go to standard out
include-graph -c configfile.txt -o outfile.dot

# You should generate the graph using graphviz/dot
# For example for the above `outfile.dot`:
dot -T svg -o outfile.svg outfile.dot

# If you install watchexec, you can setup a pipeline like:
watchexec -e txt -- "include-graph -c cfg.txt -o out.dot && dot -Tsvg -o out.svg out.dot"
```

## Configuration file format

Here is an example configuration file with comments

```txt
# Comments start with `#` and last to the end of the line

# Variables are declared first and you can nest variables
# Expansion is specifically `${name}` (this is not shell, so `$name` will not work)
SOURCE_ROOT=/some/path/to/source
OUTPUT_ROOT=${SOURCE_ROOT}/build/out

# The input section describes what files are to be parsed.
#   - what include path  should be searched for `#include "foo.h"`
#   - what files to be parsed using glob rules
input {
    # You may include a compile_commands database which will parse
    # includes (find `-I` arguments to a compiler) or sources.
    #
    # Note that compildb may not include all sources in a directory
    # in which case you should use GLOB for full coverage
    from compiledb ${OUTPUT_ROOT}/compile_commands.json load include_dirs
    from compiledb ${OUTPUT_ROOT}/other_compile_commands.json load sources
    from compiledb ${OUTPUT_ROOT}/third.json load sources, include_dirs
    
    # You may also manually include single directories
    include_dir ${SOURCE_ROOT}/includes/test
    include_dir /third/party/lib

    # Globs are generally including all files. program filters
    # out based on extensions (h, hpp, c, cpp, cxx, cc)
    glob ${SOURCE_ROOT}/src/lib1/**/*
    glob ${SOURCE_ROOT}/src/lib2/**/*
}

# The graph section defines how to setup the graph.
graph {
   # The tool works with absolute paths when parsing includes
   # the `map` section describes how to shorten typically long
   # absolute paths like `/home/user/devel/some/path/....`
   map {
      # Replace some long poath with a short prefix
      ${SOURCE_ROOT}/src/lib1 => first::
      ${SOURCE_ROOT}/src/lib2 => second::

      # This defines what items to actually keep in the output. Not
      # all includes are kept as they would be generally too large.
      #
      # Only prefix-paths are used here
      keep first::
      keep second::

      # Explicitly remove some of the kept items
      drop first::tests/
      drop second::support/library
   }

   # The group section defines how the graph should place
   # things together for easy dependency view.
   # Group logic:
   #   - they must be in order of application
   #   - first group wins (a file belongs to one group only)
   group {
      # This loads build targets from GN:
      #  - compile_root is where the `gn` too will be pointed to
      #  - target defines what GN target to grab `sources` from
      #  - sources will translate gn paths of `//foo` into absolute
      #    system paths
      gn root ${OUTPUT_ROOT} target //src/app/* sources ${SOURCE_ROOT}

      # Groups can be manually defined to group some files
      manual group-name-here {
         # Grouping is done by mapped names
         first::platform/Header.h
         first::platform/Src.cpp
         first::Something.cc
       }

      # Manual groups may have an optional color
      manual group-name-here color lightblue {
        # Grouping is done by mapped names
        first::some/other_header.h
      }
      
      # Optional instructions to ensure headers and sources
      # are grouped together (as they are generally included in
      # the same compilation unit)
      group_source_header
   }

   # you can optionally provide instructions for edge coloring
   color edges {
     from some_group_name red

     # color may be prefixed with "bold" for a bold edge coloring
     to other_group_name bold blue
   }

   # If zoom is non-empty it generates a separate area
   # with the specified "groups" expanded and viewing individual
   # members and dependencies
   zoom {
     # The zoom takes the group name argument, which could
     # be the GN group name or the manual group name
     //src/library:support
     group-name-here
     
     # Focus prefix means to focus on determining dependencies
     # in and out of the specified group(s).
     #
     # Dependences from other zoomed items will only be show
     # if internal or they start/end in a focused zoo group
     focus: //src/library:foo
     focus: //src/something/else:else
   }
}
```

## Debug logging

The program uses [env_logger](https://docs.rs/env_logger/latest/env_logger/) for log configuration.

You can set `RUST_LOG` environment variable to control some more verbose loggin. Note that it
can be very verbose.

### Include processing

Use `RUST_LOG=include-extract=info` to print out all parsed include information

Use `RUST_LOG=full-file-list=info` to print out all files found by globbing for sources

Use `RUST_LOG=gn-path=info` to print out all files found by globbing for sources

Use `RUST_LOG=compile-db=info` to print out information parsed from the compilation database
Use `RUST_LOG=compile-db=debug` to debug parsing of the compilation database

## Releasing

Use [cargo-smart-release](https://crates.io/crates/cargo-smart-release) via
`cargo install cargo-smart-release` and then `cargo smart-release`.

Use `cargo smart-release -h` for smart release flags.
