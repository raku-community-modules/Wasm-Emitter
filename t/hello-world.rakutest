use v6.d;
use Wasm::Emitter;
use Wasm::Emitter::Expression;
use Wasm::Emitter::Function;
use Wasm::Emitter::Types;
use Test;

sub has-wasmtime() {
    so qqx/wasmtime -h/
}

sub is-wasmtime-output(Buf $wasm, $expected) {
    # Write WASM to a temporary file.
    my $temp-file = $*TMPDIR.add("raku-wasm-$*PID.wasm");
    spurt $temp-file, $wasm;
    LEAVE try unlink $temp-file;

    # Try running it.
    my $exitcode = -1;
    my $output = '';
    react {
        my $proc = Proc::Async.new('wasmtime', $temp-file);
        whenever $proc.stdout {
            $output ~= $_;
        }
        whenever $proc.stderr.lines {
            diag $_;
        }
        whenever $proc.start {
            $exitcode = .exitcode;
        }
    }

    # Analyze results.
    is $exitcode, 0, "wasmtime exitted successfully";
    if $output ~~ $expected {
        pass "Correct output";
    }
    else {
        diag "Got $output.raku()";
        flunk "Correct output";
    }
}

if has-wasmtime() {
    subtest 'Hello world' => {
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

        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-wasmtime-output $buf, "hello world\n";
    }
}
else {
    skip 'No wasmtime available to run test output; skipping';
}

done-testing;
