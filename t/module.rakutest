use v6.d;
use Wasm::Emitter;
use Wasm::Emitter::Elements;
use Wasm::Emitter::Expression;
use Wasm::Emitter::Function;
use Wasm::Emitter::Types;
use Test;

sub has-wasmtime() {
    so qqx/wasmtime -h/
}

sub is-wasm-accepted(Buf $wasm) {
    # Write WASM to a temporary file.
    my $temp-file = $*TMPDIR.add("raku-wasm-$*PID.wasm");
    spurt $temp-file, $wasm;
    LEAVE try unlink $temp-file;

    # Try running it.
    my $exitcode = -1;
    react {
        my $proc = Proc::Async.new('wasmtime', $temp-file);
        whenever $proc.stdout { #`( drop ) }
        whenever $proc.stderr.lines {
            diag $_;
        }
        whenever $proc.start {
            $exitcode = .exitcode;
        }
    }
    is $exitcode, 0, "wasmtime exitted successfully";
}

if has-wasmtime() {
    subtest 'Empty' => {
        my $emitter = Wasm::Emitter.new;
        my $buf = $emitter.assemble();
        pass 'Assembled empty module';
        is-wasm-accepted $buf;
    }

    subtest 'Function types' => {
        my $emitter = Wasm::Emitter.new;
        is $emitter.function-type(functype(resulttype(i64(), i32()), resulttype(i64()))),
            0, 'Function type interned at index 0';
        is $emitter.function-type(functype(resulttype(i32(), i32()), resulttype(i64()))),
                1, 'Different type interned at index 1';
        is $emitter.function-type(functype(resulttype(i32(), i32()), resulttype(i32()))),
                2, 'Different type interned at index 2';
        is $emitter.function-type(functype(resulttype(i32(), i32()), resulttype(i64()))),
                1, 'Interning works (1)';
        is $emitter.function-type(functype(resulttype(i64(), i32()), resulttype(i64()))),
                0, 'Interning works (2)';

        my $buf = $emitter.assemble();
        pass 'Assembled module with some function types';
        is-wasm-accepted $buf;
    }

    subtest 'Function imports' => {
        my $emitter = Wasm::Emitter.new;
        my $typeidx = $emitter.function-type:
                functype(resulttype(i32(), i32(), i32(), i32()), resulttype(i32()));
        is $emitter.import-function("wasi_unstable", "fd_write", $typeidx),
                0, 'First imported function got expected 0 index';
        is $emitter.import-function("wasi_unstable", "fd_read", $typeidx),
                1, 'Second imported function got expected 1 index';

        my $buf = $emitter.assemble();
        pass 'Assembled module with some function imports';
        is-wasm-accepted $buf;
    }

    subtest 'Declare a memory' => {
        my $emitter = Wasm::Emitter.new;
        is $emitter.memory(limitstype(0)), 0,
                'Expected index for added memory';

        my $buf = $emitter.assemble();
        pass 'Assembled module with some function imports';
        is-wasm-accepted $buf;
    }

    subtest 'Declare and export a memory' => {
        my $emitter = Wasm::Emitter.new;
        my $memory = $emitter.memory(limitstype(1));
        $emitter.export-memory("memory", $memory);

        my $buf = $emitter.assemble();
        pass 'Assembled module with some exported memory';
        is-wasm-accepted $buf;
    }

    subtest 'Declare data' => {
        my $emitter = Wasm::Emitter.new;
        $emitter.memory(limitstype(1));
        is $emitter.passive-data("hello world\n".encode('utf-8')), 0,
                'First (passive) data declaration at index zero';
        my $expression = Wasm::Emitter::Expression.new;
        $expression.i32-const(8);
        is $emitter.active-data("hello world\n".encode('utf-8'), $expression), 1,
                'Second (active) data declaration at index zero';

        my $buf = $emitter.assemble();
        pass 'Assembled module with data section';
        is-wasm-accepted $buf;
    }

    subtest 'Function declaration' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type(functype(resulttype(), resulttype(i32())));
        my $expression = Wasm::Emitter::Expression.new;
        $expression.i32-const(42);
        is $emitter.function(Wasm::Emitter::Function.new(:$type-index, :$expression)),
                0, 'Correct index for first added function';

        my $buf = $emitter.assemble();
        pass 'Assembled module with function and code sections';
        is-wasm-accepted $buf;
    }

    subtest 'Function export' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type(functype(resulttype(), resulttype(i32())));
        my $expression = Wasm::Emitter::Expression.new;
        $expression.i32-const(42);
        my $func-index = $emitter.function(Wasm::Emitter::Function.new(:$type-index, :$expression));
        $emitter.export-function('answer', $func-index);

        my $buf = $emitter.assemble();
        pass 'Assembled module with function export';
        is-wasm-accepted $buf;
    }

    subtest 'Declare and export globals' => {
        my $emitter = Wasm::Emitter.new;
        my $expression = Wasm::Emitter::Expression.new;
        $expression.i64-const(42);
        is $emitter.global(globaltype(i64()), $expression), 0,
                'Got expected index for first added global';
        is $emitter.global(globaltype(i64(), :mutable), $expression), 1,
                'Got expected index for second added global';
        $emitter.export-global('foo', $emitter.global(globaltype(i64()), $expression));

        my $buf = $emitter.assemble();
        pass 'Assembled module with some globals';
        is-wasm-accepted $buf;
    }

    subtest 'Declare and export tables' => {
        my $emitter = Wasm::Emitter.new;
        is $emitter.table(tabletype(limitstype(8, 8), funcref())), 0,
            'First table got expected index';
        is $emitter.table(tabletype(limitstype(8), externref())), 1,
                'Second table got expected index';
        $emitter.export-table('funcymcfuncface', $emitter.table(tabletype(limitstype(4), funcref)));

        my $buf = $emitter.assemble();
        pass 'Assembled module with some tables';
        is-wasm-accepted $buf;
    }

    subtest 'Declare elements segments' => {
        # Set up some tables and functions for use in the test.
        my $emitter = Wasm::Emitter.new;
        my $table-a = $emitter.table(tabletype(limitstype(8), funcref()));
        my $table-b = $emitter.table(tabletype(limitstype(8), funcref()));
        my @init = do for ^4 {
            my $type-index = $emitter.function-type(functype(resulttype(), resulttype(i32())));
            my $expression = Wasm::Emitter::Expression.new;
            $expression.i32-const(42);
            my $func-index = $emitter.function(Wasm::Emitter::Function.new(:$type-index, :$expression));
            my $init-expression = Wasm::Emitter::Expression.new;
            $init-expression.ref-func($func-index);
            $init-expression
        }

        # Add various kinds of element section.
        is $emitter.elements(Wasm::Emitter::Elements::Declarative.new(:type(funcref()), :@init)),
                0, 'Correct first elements index';
        is $emitter.elements(Wasm::Emitter::Elements::Passive.new(:type(funcref()), :@init)),
                1, 'Correct second elements index';
        my $offset = Wasm::Emitter::Expression.new;
        $offset.i32-const(0);
        is $emitter.elements(Wasm::Emitter::Elements::Active.new(:type(funcref()), :@init, :0table-index, :$offset)),
                2, 'Correct third elements index';
        is $emitter.elements(Wasm::Emitter::Elements::Active.new(:type(funcref()), :@init, :1table-index, :$offset)),
                3, 'Correct fourth elements index';

        my $buf = $emitter.assemble();
        pass 'Assembled module with a variety of elements segments';
        is-wasm-accepted $buf;
    }

    subtest 'Other imports' => {
        my $emitter = Wasm::Emitter.new;
        is $emitter.import-table("mod", "tab1", tabletype(limitstype(8), funcref())),
                0, 'First table import gets expected index';
        is $emitter.import-table("mod", "tab2", tabletype(limitstype(8), funcref())),
                1, 'Second table import gets expected index';
        is $emitter.import-memory("mod", "mem1", limitstype(0)),
                0, 'First memory import gets expected index';
        is $emitter.import-memory("mod", "mem2", limitstype(8)),
                1, 'Second memory import gets expected index';
        is $emitter.import-global("mod", "global1", globaltype(i64())),
                0, 'First global import gets expected index';
        is $emitter.import-global("mod", "global2", globaltype(i64(), :mutable)),
                1, 'Second global import gets expected index';

        $emitter.assemble();
        pass 'Assembled module with some function imports';
        # No obviously available real imports to test it with against runtime
    }

    subtest 'Imports affect offsets' => {
        my $emitter = Wasm::Emitter.new;
        $emitter.import-table("mod", "tab1", tabletype(limitstype(8), funcref()));
        $emitter.import-memory("mod", "mem1", limitstype(0));
        $emitter.import-global("mod", "global1", globaltype(i64()));

        is $emitter.table(tabletype(limitstype(8), funcref())), 1,
            'Table indices account for table imports';
        is $emitter.memory(limitstype(64)), 1,
            'Memory indices account for memory imports';
        my $expression = Wasm::Emitter::Expression.new;
        $expression.i64-const(42);
        is $emitter.global(globaltype(i64()), $expression), 1,
            'Global indices account for global imports';
    }

    subtest 'Cannot add imports after declarations of the same kind' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type(functype(resulttype(), resulttype(i64())));
        my $expression = Wasm::Emitter::Expression.new;
        $expression.i64-const(42);
        $emitter.function(Wasm::Emitter::Function.new(:$type-index, :$expression));
        $emitter.table(tabletype(limitstype(8), funcref()));
        $emitter.memory(limitstype(64));
        $emitter.global(globaltype(i64()), $expression);

        dies-ok { $emitter.import-function('mod', 'func', $type-index) },
            'Cannot import function after declaring one';
        dies-ok { $emitter.import-table("mod", "tab1", tabletype(limitstype(8), funcref())) },
            'Cannot import table after declaring one';
        dies-ok { $emitter.import-memory("mod", "mem1", limitstype(0)); },
            'Cannot import memory after declaring one';
        dies-ok { $emitter.import-global("mod", "global1", globaltype(i64())) },
            'Cannot import global after declaring one';
    }
}
else {
    skip 'No wasmtime available to run test output; skipping';
}

done-testing;
