[![Actions Status](https://github.com/lizmat/Wasm-Emitter/actions/workflows/test.yml/badge.svg)](https://github.com/lizmat/Wasm-Emitter/actions)

NAME
====

Wasm::Emitter - Emit the WebAssembly binary format

SYNOPSIS
========

```raku
use Wasm::Emitter;
```

DESCRIPTION
===========

A Raku module to emit the [WebAssembly](https://webassembly.org/) binary format.

Example
=======

This emits the "hello world" WebAssembly program using WASI (the WebAssembly System Interface) to provide I/O functions. 

```raku
# Create an instance of the emitter, which can emit a module.
my $emitter = Wasm::Emitter.new;

# Import fd_write
my $fd-write-type = $emitter.function-type:
        functype(resulttype(i32(), i32(), i32(), i32()), resulttype(i32()));
my $fd-write-index = $emitter.import-function("wasi_unstable", "fd_write", $fd-write-type);

# Declare and export a memory.
$emitter.export-memory("memory", $emitter.memory(limitstype(1)));

# Write 'hello world\n' to memory at an offset of 8 bytes
my $offset-expression = Wasm::Emitter::Expression.new;
$offset-expression.i32-const(8);
$emitter.active-data("hello world\n".encode('utf-8'), $offset-expression);

# Generate code to call fd_write.
my $code = Wasm::Emitter::Expression.new;
given $code {
    # (i32.store (i32.const 0) (i32.const 8))
    .i32-const(0);
    .i32-const(8);
    .i32-store;
    # (i32.store (i32.const 4) (i32.const 12))
    .i32-const(4);
    .i32-const(12);
    .i32-store;
    # (call $fd_write
    #   (i32.const 1) ;; file_descriptor - 1 for stdout
    #   (i32.const 0) ;; *iovs - The pointer to the iov array, which is stored at memory location 0
    #   (i32.const 1) ;; iovs_len - We're printing 1 string stored in an iov - so one.
    #   (i32.const 20) ;; nwritten - A place in memory to store the number of bytes written
    # )
    .i32-const(1);
    .i32-const(0);
    .i32-const(1);
    .i32-const(20);
    .call($fd-write-index);
    # Drop return value
    .drop;
}

# Declare and export the start function.
my $start-type = $emitter.function-type: functype(resulttype(), resulttype());
my $start-func-index = $emitter.function: Wasm::Emitter::Function.new:
        :type-index($start-type), :expression($code);
$emitter.export-function('_start', $start-func-index);

# Assemble and write to a file, which can be run by, for example, wasmtime.
my $buf = $emitter.assemble();
spurt 'hello.wasm', $buf;
```

Functionality
=============

To the best of my knowledge, this covers all of the WebAssembly 2.0 specified features except the vector instructions. None of the proposed specifications are currently implemented (although some of them are liable to attract my attention ahead of the vector instructions).

Support policy
==============

This is a module developed for personal interest. I'm sharing it in case it's fun or useful to anybody else, not because I want yet another open source project to maintain. PRs that include tests and are provided in an easy to review form will likely be merged quite quickly, so long as no major flaws are noticed.

API Documentation
=================

class Wasm::Emitter
-------------------

Emitter for a binary Wasm module. An instance of this represents a module. Make the various declarations, and then call C<assemble> to produce a C<Buf> with the WebAssembly.

### has Positional[Wasm::Emitter::Types::FunctionType] @!function-types

Function types, all distinct.

### has Positional[Wasm::Emitter::FunctionImport] @!function-imports

Function imports.

### has Positional[Wasm::Emitter::TableImport] @!table-imports

Table imports.

### has Positional[Wasm::Emitter::MemoryImport] @!memory-imports

Memory imports.

### has Positional[Wasm::Emitter::GlobalImport] @!global-imports

Global imports.

### has Positional[Wasm::Emitter::Types::TableType] @!tables

Declared tables, with their table types.

### has Positional[Wasm::Emitter::Types::LimitType] @!memories

Declared memories, with their limits.

### has Positional[Wasm::Emitter::Export] @!exports

Declared exports.

### has Positional[Wasm::Emitter::Data] @!data

Declared data sections.

### has Positional[Wasm::Emitter::Global] @!globals

Declared globals.

### has Positional[Wasm::Emitter::Elements] @!elements

Declared elements.

### has Positional[Wasm::Emitter::Function] @!functions

Declared functions.

### method function-type

```raku
method function-type(
    Wasm::Emitter::Types::FunctionType $type
) returns Int
```

Returns a type index for a function type. If the function type was already registered, returns the existing index; failing that, adds it under a new index.

### method import-function

```raku
method import-function(
    Str $module,
    Str $name,
    Int $type-index
) returns Int
```

Add a function import.

### method import-table

```raku
method import-table(
    Str $module,
    Str $name,
    Wasm::Emitter::Types::TableType $table-type
) returns Int
```

Add a table import.

### method import-memory

```raku
method import-memory(
    Str $module,
    Str $name,
    Wasm::Emitter::Types::LimitType $memory-type
) returns Int
```

Add a memory import.

### method import-global

```raku
method import-global(
    Str $module,
    Str $name,
    Wasm::Emitter::Types::GlobalType $global-type
) returns Int
```

Add a global import.

### method table

```raku
method table(
    Wasm::Emitter::Types::TableType $table-type
) returns Int
```

Declare a table.

### method memory

```raku
method memory(
    Wasm::Emitter::Types::LimitType $limits
) returns Int
```

Add a declaration of a memory.

### method export-function

```raku
method export-function(
    Str $name,
    Int $function-index
) returns Nil
```

Export a function.

### method export-memory

```raku
method export-memory(
    Str $name,
    Int $memory-index
) returns Nil
```

Export a memory.

### method export-global

```raku
method export-global(
    Str $name,
    Int $global-index
) returns Nil
```

Export a global.

### method export-table

```raku
method export-table(
    Str $name,
    Int $table-index
) returns Nil
```

Export a table.

### method passive-data

```raku
method passive-data(
    Blob $data
) returns Int
```

Declare a passive data section.

### method active-data

```raku
method active-data(
    Blob $data,
    Wasm::Emitter::Expression $offset
) returns Int
```

Declare an active data section.

### method global

```raku
method global(
    Wasm::Emitter::Types::GlobalType $type,
    Wasm::Emitter::Expression $init
) returns Int
```

Declare a global.

### method elements

```raku
method elements(
    Wasm::Emitter::Elements $elements
) returns Int
```

Declare an elements section.

### method function

```raku
method function(
    Wasm::Emitter::Function $function
) returns Int
```

Declare a function.

### method assemble

```raku
method assemble() returns Buf
```

Assemble the produced declarations into a final output.

AUTHOR
======

Jonathan Worthington

COPYRIGHT AND LICENSE
=====================

Copyright 2022 - 2024 Jonathan Worthington

Copyright 2024 Raku Community

This library is free software; you can redistribute it and/or modify it under the Artistic License 2.0.

