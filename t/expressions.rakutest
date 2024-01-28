use v6.d;
use Wasm::Emitter;
use Wasm::Emitter::Expression;
use Wasm::Emitter::Function;
use Wasm::Emitter::Types;
use Test;

sub has-wasmtime() {
    so qqx/wasmtime -h/
}

sub is-function-output(Buf $wasm, @args, $expected, :$function = 'test') {
    # Write WASM to a temporary file.
    my $temp-file = $*TMPDIR.add("raku-wasm-$*PID.wasm");
    spurt $temp-file, $wasm;
    LEAVE try unlink $temp-file;

    # Try running it.
    my $exitcode = -1;
    my $output = '';
    react {
        my $proc = Proc::Async.new('wasmtime', $temp-file, '--invoke' , $function, '--', @args);
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
    $output .= trim;
    if $output ~~ $expected {
        pass "Correct output";
    }
    else {
        diag "Got $output.raku()";
        flunk "Correct output";
    }
}

sub test-nullary(Wasm::Emitter::Expression $expression, Wasm::Emitter::Types::ValueType $type, $expected) {
    my $emitter = Wasm::Emitter.new;
    my $type-index = $emitter.function-type: functype(resulttype(), resulttype($type));
    my $func-index = $emitter.function: Wasm::Emitter::Function.new(:$type-index, :$expression);
    $emitter.export-function('test', $func-index);
    my $buf = $emitter.assemble();
    pass 'Assembled module';
    is-function-output $buf, [], $expected;
}

if has-wasmtime() {
    for i32(), 'i32-const', i64(), 'i64-const' -> $type, $op {
        subtest "$op" => {
            my $expression = Wasm::Emitter::Expression.new;
            $expression."$op"(12345);
            test-nullary $expression, $type, '12345';
        }
    }

    for f32(), 'f32-const', f64(), 'f64-const' -> $type, $op {
        subtest "$op" => {
            my $expression = Wasm::Emitter::Expression.new;
            $expression."$op"(12.45e0);
            test-nullary $expression, $type, '12.45';
        }
    }

    for 'f32-', 'f64-' -> $op-base {
        for 'eq', 0, 'ne', 1, 'lt', 0, 'le', 0, 'gt', 1, 'ge', 1 -> $op, $expected {
            subtest "$op-base$op" => {
                my $expression = Wasm::Emitter::Expression.new;
                $expression."{ $op-base }const"(10.9e0);
                $expression."{ $op-base }const"(5.2e0);
                $expression."$op-base$op"();
                test-nullary $expression, i32(), ~$expected;
            }
        }
    }

    for 'i32-', 'i64-' -> $op-base {
        for 'eq', 0, 'ne', 1, 'lt-s', 0, 'lt-u', 1, 'le-s', 0, 'le-u', 1,
                'gt-s', 1, 'gt-u', 0, 'ge-s', 1, 'ge-u', 0 -> $op, $expected {
            subtest "$op-base$op" => {
                my $expression = Wasm::Emitter::Expression.new;
                $expression."{ $op-base }const"(10);
                $expression."{ $op-base }const"(-5);
                $expression."$op-base$op"();
                test-nullary $expression, i32(), ~$expected;
            }
        }
        for 0, 1, 1, 0 -> $value, $expected {
            subtest "{$op-base}eqz" => {
                my $expression = Wasm::Emitter::Expression.new;
                $expression."{ $op-base }const"($value);
                $expression."{$op-base}eqz"();
                test-nullary $expression, i32(), ~$expected;
            }
        }
    }

    for 'clz', 5, 'ctz', 6, 'popcnt', 7 -> $op, $expected {
        subtest "i32-$op" => {
            my $expression = Wasm::Emitter::Expression.new;
            $expression."i32-const"(0b0000_0100_0011_0010_0000_0011_0100_0000);
            $expression."i32-$op"();
            test-nullary $expression, i32(), $expected;
        }
        subtest "i64-$op" => {
            my $expression = Wasm::Emitter::Expression.new;
            $expression."i64-const"(0b0000_0100_0000_0000_0011_0000_0010_0000_0000_0000_0000_0000_0011_0000_0100_0000);
            $expression."i64-$op"();
            test-nullary $expression, i64(), $expected;
        }
    }

    for i32(), 'i32-', i64(), 'i64-' -> $type, $op-base {
        for 'add', 20, 'sub', 10, 'mul', 75, 'div-s', 3, 'div-u', 3,
                'rem-s', 0, 'rem-u', 0 -> $op, $expected {
            subtest "$op-base$op" => {
                my $expression = Wasm::Emitter::Expression.new;
                $expression."{ $op-base }const"(15);
                $expression."{ $op-base }const"(5);
                $expression."$op-base$op"();
                test-nullary $expression, $type, $expected;
            }
        }
    }

    for i32(), 'i32-', i64(), 'i64-' -> $type, $op-base {
        for 'and', 0b0010, 'or', 0b1110, 'xor', 0b1100 -> $op, $expected {
            subtest "$op-base$op" => {
                my $expression = Wasm::Emitter::Expression.new;
                $expression."{ $op-base }const"(0b1010);
                $expression."{ $op-base }const"(0b0110);
                $expression."$op-base$op"();
                test-nullary $expression, $type, $expected;
            }
        }
    }

    for i32(), 'i32-', i64(), 'i64-' -> $type, $op-base {
        for 'shl', 0b10101101000, 'shr-s', 0b10101, 'shr-u', 0b10101 -> $op, $expected {
            subtest "$op-base$op" => {
                my $expression = Wasm::Emitter::Expression.new;
                $expression."{ $op-base }const"(0b10101101);
                $expression."{ $op-base }const"(3);
                $expression."$op-base$op"();
                test-nullary $expression, $type, $expected;
            }
        }
    }

    for i32(), 'i32-', i64(), 'i64-' -> $type, $op-base {
        for 'rotl', 0b1101_1000, 'rotr', 0b0011_0110  -> $op, $expected {
            subtest "$op-base$op" => {
                my $expression = Wasm::Emitter::Expression.new;
                $expression."{ $op-base }const"(0b0110_1100);
                $expression."{ $op-base }const"(1);
                $expression."$op-base$op"();
                test-nullary $expression, $type, $expected;
            }
        }
    }

    for f32(), 'f32-', f64(), 'f64-' -> $type, $op-base {
        for 'abs', '30.25', 'neg', '-30.25', 'ceil', '31', 'floor', '30',
                'trunc', '30', 'nearest', '30', 'sqrt', '5.5' -> $op, $expected {
            subtest "$op-base$op" => {
                my $expression = Wasm::Emitter::Expression.new;
                $expression."{ $op-base }const"(30.25e0);
                $expression."$op-base$op"();
                test-nullary $expression, $type, $expected;
            }
        }
    }

    for f32(), 'f32-', f64(), 'f64-' -> $type, $op-base {
        for 'add', '7', 'sub', '14', 'mul', '-36.75', 'div', '-3',
                'min', '-3.5', 'max', '10.5', 'copysign', '-10.5' -> $op, $expected {
            subtest "$op-base$op" => {
                my $expression = Wasm::Emitter::Expression.new;
                $expression."{ $op-base }const"(10.5e0);
                $expression."{ $op-base }const"(-3.5e0);
                $expression."$op-base$op"();
                test-nullary $expression, $type, $expected;
            }
        }
    }

    subtest 'Locals and instructions (get/set/tee)' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        my $local-id-a = $function.local(i32());
        my $local-id-b = $function.local(i32());
        $expression.i32-const(99);
        $expression.local-set($local-id-a);
        $expression.local-get($local-id-a);
        $expression.local-tee($local-id-b);
        $expression.drop;
        $expression.local-get($local-id-b);
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], '99';
    }

    subtest 'Parameters' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(i32(), i32()), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :2parameters, :$expression);
        my $local-id = $function.local(i32());
        is $local-id, 2, 'Locals numbered from parameter count';
        $expression.local-get(0);
        $expression.local-get(1);
        $expression.i32-sub();
        $expression.local-set($local-id);
        $expression.local-get($local-id);
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [49, 7], '42';
    }

    subtest 'nop, return, and unreachable' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        $expression.nop();
        $expression.i32-const(101);
        $expression.nop();
        $expression.return();
        $expression.unreachable();
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], '101';
    }

    subtest 'if' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(i32()), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :1parameters, :$expression);
        $expression.local-get(0);
        $expression.if: {
            $expression.i32-const(99);
            $expression.return;
        }
        $expression.i32-const(100);
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [1], '99';
        is-function-output $buf, [0], '100';
    }

    subtest 'if' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(i32()), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :1parameters, :$expression);
        $expression.local-get(0);
        $expression.if: :blocktype(i32()),
                {
                    $expression.i32-const(25);
                },
                {
                    $expression.i32-const(75);
                };
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [1], '25';
        is-function-output $buf, [0], '75';
    }

    subtest 'block/br' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        $expression.block: :blocktype(i32()), {
            $expression.i32-const(66);
            $expression.br(0);
            $expression.drop();
            $expression.i32-const(69);
        }
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], '66';
    }

    subtest 'block/br_if' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(i32()), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index :1parameters, :$expression);
        $expression.block: :blocktype(i32()), {
            $expression.i32-const(66);
            $expression.local-get(0);
            $expression.br-if(0);
            $expression.drop();
            $expression.i32-const(69);
        }
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [1], '66';
        is-function-output $buf, [0], '69';
    }

    subtest 'loop' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(i32()), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index :1parameters, :$expression);
        my $res = $function.local(i32());
        $expression.i32-const(1);
        $expression.local-set($res);
        $expression.local-get(0);
        $expression.if: {
            $expression.loop: {
                $expression.local-get(0);
                $expression.local-get($res);
                $expression.i32-mul();
                $expression.local-set($res);
                $expression.local-get(0);
                $expression.i32-const(1);
                $expression.i32-sub();
                $expression.local-tee(0);
                $expression.br-if();
            }
        }
        $expression.local-get($res);
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [0], '1';
        is-function-output $buf, [1], '1';
        is-function-output $buf, [2], '2';
        is-function-output $buf, [3], '6';
        is-function-output $buf, [4], '24';
        is-function-output $buf, [5], '120';
    }

    subtest 'br_table' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(i32()), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index :1parameters, :$expression);
        $expression.block: {
            $expression.block: {
                $expression.block: {
                    $expression.local-get(0);
                    $expression.br-table([0, 1], 2)
                }
                $expression.i32-const(100);
                $expression.return();
            }
            $expression.i32-const(200);
            $expression.return();
        }
        $expression.i32-const(300);
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [0], '100';
        is-function-output $buf, [1], '200';
        is-function-output $buf, [2], '300';
        is-function-output $buf, [3], '300';
        is-function-output $buf, [4], '300';
    }

    subtest 'select (no type)' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(i32()), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index :1parameters, :$expression);
        $expression.i32-const(55);
        $expression.i32-const(66);
        $expression.local-get(0);
        $expression.select();
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [0], '66';
        is-function-output $buf, [1], '55';
    }

    subtest 'select (type)' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(i32()), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index :1parameters, :$expression);
        $expression.i32-const(55);
        $expression.i32-const(66);
        $expression.local-get(0);
        $expression.select(i32());
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [0], '66';
        is-function-output $buf, [1], '55';
    }

    sub coercion-test($op, $in-type, $out-type, *@cases, :$const-ins) {
        for @cases -> $case {
            my $emitter = Wasm::Emitter.new;
            my $in-resulttype = $const-ins ?? resulttype() !! resulttype($in-type);
            my $type-index = $emitter.function-type: functype($in-resulttype, resulttype($out-type));
            my $expression = Wasm::Emitter::Expression.new;
            my $function = Wasm::Emitter::Function.new(:$type-index :parameters($const-ins ?? 0 !! 1), :$expression);
            if $const-ins {
                $expression."$const-ins"($case.key);
            }
            else {
                $expression.local-get(0);
            }
            $expression."$op"();
            $emitter.export-function('test', $emitter.function($function));
            my $buf = $emitter.assemble();
            pass 'Assembled module';
            is-function-output $buf, $const-ins ?? [] !! [$case.key], $case.value;
        }
    }

    subtest 'i32.wrap_i64' => {
        coercion-test 'i32-wrap-i64', i64(), i32(), 0xFFFFFF => 0xFFFFFF, 0xFFFFFFFF => -1;
    }

    subtest 'i64.extend_*' => {
        coercion-test 'i64-extend-i32-s', i32(), i64(), 10 => 10, -9 => -9;
        coercion-test 'i64-extend-i32-u', i32(), i64(), 10 => 10, -9 => 4294967287;
    }

    subtest 'i32.trunc_* and i64.trunc_*' => {
        coercion-test 'i32-trunc-f32-s', :const-ins<f32-const>, f32(), i32(), 10.55e0 => 10, -90.2e0 => -90;
        coercion-test 'i32-trunc-f32-u', :const-ins<f32-const>, f32(), i32(), 10.55e0 => 10;
        coercion-test 'i32-trunc-f64-s', :const-ins<f64-const>, f64(), i32(), 10.55e0 => 10, -90.2e0 => -90;
        coercion-test 'i32-trunc-f64-u', :const-ins<f64-const>, f64(), i32(), 10.55e0 => 10;
        coercion-test 'i64-trunc-f32-s', :const-ins<f32-const>, f32(), i64(), 10.55e0 => 10, -90.2e0 => -90;
        coercion-test 'i64-trunc-f32-u', :const-ins<f32-const>, f32(), i64(), 10.55e0 => 10;
        coercion-test 'i64-trunc-f64-s', :const-ins<f64-const>, f64(), i64(), 10.55e0 => 10, -90.2e0 => -90;
        coercion-test 'i64-trunc-f64-u', :const-ins<f64-const>, f64(), i64(), 10.55e0 => 10;
    }

    subtest 'f32.convert_* and f64.convert_*' => {
        coercion-test 'f32-convert-i32-s', i32(), f32(), 64 => 64, -4 => -4;
        coercion-test 'f32-convert-i32-u', i32(), f32(), 64 => 64, -4 => 4294967300;
        coercion-test 'f32-convert-i64-s', i64(), f32(), 64 => 64, -4 => -4;
        coercion-test 'f32-convert-i64-u', i64(), f32(), 64 => 64, -4 => 18446744000000000000;
        coercion-test 'f64-convert-i32-s', i32(), f64(), 64 => 64, -4 => -4;
        coercion-test 'f64-convert-i32-u', i32(), f64(), 64 => 64, -4 => 4294967292;
        coercion-test 'f64-convert-i64-s', i64(), f64(), 64 => 64, -4 => -4;
        coercion-test 'f64-convert-i64-u', i64(), f64(), 64 => 64, -4 => 18446744073709552000;
    }

    subtest 'f32.demote_f64 and f64.promote_f32' => {
        coercion-test 'f32-demote-f64', :const-ins<f64-const>, f64(), f32(),
                -42e0 => -42, 18446744073709552000e0 => 18446744000000000000;
        coercion-test 'f64-promote-f32', :const-ins<f32-const>, f32(), f64(),
                -42e0 => -42, 18446744000000000000e0 => 18446744073709552000;
    }

    subtest '*.reinterpret_* instructions' => {
        coercion-test 'i32-reinterpret-f32', :const-ins<f32-const>, f32(), i32(),
                42.5e0 => 1110048768;
        coercion-test 'i64-reinterpret-f64', :const-ins<f64-const>, f64(), i64(),
                42.5e0 => 4631178160564600832;
        coercion-test 'f32-reinterpret-i32', i32(), f32(), 1110048768 => 42.5;
        coercion-test 'f64-reinterpret-i64', i64(), f64(), 4631178160564600832 => 42.5;
    }

    subtest '*.extend8_s, *.extend16_s, and i64.extend32_s' => {
        coercion-test 'i32-extend8-s', i32(), i32(), 0xFF => -1;
        coercion-test 'i32-extend16-s', i32(), i32(), 0xFFFF => -1;
        coercion-test 'i64-extend8-s', i64(), i64(), 0xFF => -1;
        coercion-test 'i64-extend16-s', i64(), i64(), 0xFFFF => -1;
        coercion-test 'i64-extend32-s', i64(), i64(), 0xFFFFFFFF => -1;
    }

    subtest 'i32.trunc_* and i64.trunc_*' => {
        coercion-test 'i32-trunc-sat-f32-s', :const-ins<f32-const>, f32(), i32(),
                10.55e0 => 10, -90.2e0 => -90, 0xFFFFFFFFFF.Num => 0x7FFFFFFF;
        coercion-test 'i32-trunc-sat-f32-u', :const-ins<f32-const>, f32(), i32(),
                10.55e0 => 10, -90.2e0 => 0, 0xFFFFFFFFFF.Num => -1; # Output treated as signed by printer
        coercion-test 'i32-trunc-sat-f64-s', :const-ins<f64-const>, f64(), i32(),
                10.55e0 => 10, -90.2e0 => -90, 0xFFFFFFFFFF.Num => 0x7FFFFFFF;
        coercion-test 'i32-trunc-sat-f64-u', :const-ins<f64-const>, f64(), i32(),
                10.55e0 => 10, -90.2e0 => 0, 0xFFFFFFFFFF.Num => -1; # Output treated as signed by printer
        coercion-test 'i64-trunc-sat-f32-s', :const-ins<f32-const>, f32(), i64(),
                10.55e0 => 10, -90.2e0 => -90, 0xFFFFFFFFFFFFFFFFFF.Num => 0x7FFFFFFFFFFFFFFF;
        coercion-test 'i64-trunc-sat-f32-u', :const-ins<f32-const>, f32(), i64(),
                10.55e0 => 10, -90.2e0 => 0, 0xFFFFFFFFFFFFFFFFFF.Num => -1;  # Output treated as signed by printer
        coercion-test 'i64-trunc-sat-f64-s', :const-ins<f64-const>, f64(), i64(),
                10.55e0 => 10, -90.2e0 => -90, 0xFFFFFFFFFFFFFFFFFF.Num => 0x7FFFFFFFFFFFFFFF;
        coercion-test 'i64-trunc-sat-f64-u', :const-ins<f64-const>, f64(), i64(),
                10.55e0 => 10, -90.2e0 => 0, 0xFFFFFFFFFFFFFFFFFF.Num => -1;  # Output treated as signed by printer
    }


    for 'i32-', i32(), 'i64-', i64(), 'f32-', f32(), 'f64-', f64() -> $prefix, $type {
        subtest "{$prefix}load and {$prefix}store" => {
            my $emitter = Wasm::Emitter.new;
            $emitter.memory(limitstype(1));
            my $type-index = $emitter.function-type: functype(resulttype(), resulttype($type));
            my $expression = Wasm::Emitter::Expression.new;
            my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
            $expression.i32-const(8);
            $expression."{$prefix}const"($prefix.starts-with('i') ?? 42 !! 42e0);
            $expression."{$prefix}store"();
            $expression.i32-const(8);
            $expression."{$prefix}load"();
            $emitter.export-function('test', $emitter.function($function));
            my $buf = $emitter.assemble();
            pass 'Assembled module';
            is-function-output $buf, [], 42;
        }
    }

    for '8-s', -1, '8-u', 255, '16-s', -1, '16-u', 65535 -> $suffix, $expected {
        subtest "i32.load$suffix" => {
            my $emitter = Wasm::Emitter.new;
            $emitter.memory(limitstype(1));
            my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
            my $expression = Wasm::Emitter::Expression.new;
            my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
            $expression.i32-const(4);
            $expression.i32-const(0xFFFFFF);
            $expression.i32-store();
            $expression.i32-const(4);
            $expression."i32-load$suffix"();
            $emitter.export-function('test', $emitter.function($function));
            my $buf = $emitter.assemble();
            pass 'Assembled module';
            is-function-output $buf, [], $expected;
        }
    }

    for '8-s', -1, '8-u', 255, '16-s', -1, '16-u', 65535, '32-s', -1, '32-u', 4294967295 -> $suffix, $expected {
        subtest "i64.load$suffix" => {
            my $emitter = Wasm::Emitter.new;
            $emitter.memory(limitstype(1));
            my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i64()));
            my $expression = Wasm::Emitter::Expression.new;
            my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
            $expression.i32-const(4);
            $expression.i64-const(0xFFFFFFFFFF);
            $expression.i64-store();
            $expression.i32-const(4);
            $expression."i64-load$suffix"();
            $emitter.export-function('test', $emitter.function($function));
            my $buf = $emitter.assemble();
            pass 'Assembled module';
            is-function-output $buf, [], $expected;
        }
    }

    for '8', 0x0000FF00, '16', 0x00FFFF00 -> $suffix, $expected {
        subtest "i32.store$suffix" => {
            my $emitter = Wasm::Emitter.new;
            $emitter.memory(limitstype(1));
            my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
            my $expression = Wasm::Emitter::Expression.new;
            my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
            $expression.i32-const(1);
            $expression.i32-const(0xFFFFFF);
            $expression."i32-store$suffix"(:0align);
            $expression.i32-const(0);
            $expression.i32-load();
            $emitter.export-function('test', $emitter.function($function));
            my $buf = $emitter.assemble();
            pass 'Assembled module';
            is-function-output $buf, [], $expected;
        }
    }

    for '8', 0x00000000_0000FF00, '16', 0x00000000_00FFFF00, '32', 0x000000FF_FFFFFF00 -> $suffix, $expected {
        subtest "i64.store$suffix" => {
            my $emitter = Wasm::Emitter.new;
            $emitter.memory(limitstype(1));
            my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i64()));
            my $expression = Wasm::Emitter::Expression.new;
            my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
            $expression.i32-const(1);
            $expression.i64-const(0xFFFFFFFFFF);
            $expression."i64-store$suffix"(:0align);
            $expression.i32-const(0);
            $expression.i64-load();
            $emitter.export-function('test', $emitter.function($function));
            my $buf = $emitter.assemble();
            pass 'Assembled module';
            is-function-output $buf, [], $expected;
        }
    }

    subtest 'memory.size' => {
        my $emitter = Wasm::Emitter.new;
        $emitter.memory(limitstype(4));
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        $expression.memory-size();
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], 4;
    }

    subtest 'memory.grow' => {
        my $emitter = Wasm::Emitter.new;
        $emitter.memory(limitstype(4));
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        $expression.i32-const(2);
        $expression.memory-grow();
        $expression.drop();
        $expression.memory-size();
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], 6;
    }

    subtest 'memory.fill, memory.copy' => {
        my $emitter = Wasm::Emitter.new;
        $emitter.memory(limitstype(1));
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i64()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        $expression.i32-const(1);
        $expression.i32-const(0xFF);
        $expression.i32-const(2);
        $expression.memory-fill();
        $expression.i32-const(4);
        $expression.i32-const(0);
        $expression.i32-const(4);
        $expression.memory-copy();
        $expression.i32-const(0);
        $expression.i64-load();
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], 0x00FFFF00_00FFFF00;
    }

    subtest 'memory.init, data.drop' => {
        my $emitter = Wasm::Emitter.new;
        $emitter.memory(limitstype(1));
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        my $data-idx = $emitter.passive-data(Buf.new(0xFE, 0xCA));
        $expression.i32-const(1);
        $expression.i32-const(0);
        $expression.i32-const(2);
        $expression.memory-init($data-idx);
        $expression.data-drop($data-idx);
        $expression.i32-const(0);
        $expression.i32-load();
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], 0x00CAFE00;
    }

    subtest 'global.get, global.set' => {
        my $emitter = Wasm::Emitter.new;
        my $init-expression = Wasm::Emitter::Expression.new;
        $init-expression.i32-const(2);
        my $global-idx = $emitter.global(globaltype(i32(), :mutable), $init-expression);
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        $expression.global-get($global-idx);
        $expression.i32-const(3);
        $expression.global-set($global-idx);
        $expression.global-get($global-idx);
        $expression.i32-mul();
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], 2 * 3;
    }

    subtest 'ref.null and ref.is_null' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        $expression.ref-null(funcref());
        $expression.ref-is-null();
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], 1;
    }

    subtest 'ref.func and ref.is_null' => {
        my $emitter = Wasm::Emitter.new;
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        my $func-idx = $emitter.function($function);
        $expression.ref-func($func-idx);
        $expression.ref-is-null();
        $emitter.export-function('test', $func-idx);
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], 0;
    }

    subtest 'table.get' => {
        my $emitter = Wasm::Emitter.new;
        my $table-idx = $emitter.table(tabletype(limitstype(8), externref()));
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        $expression.i32-const(0);
        $expression.table-get($table-idx);
        $expression.ref-is-null();
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], 1;
    }

    subtest 'table.set' => {
        my $emitter = Wasm::Emitter.new;
        my $table-idx = $emitter.table(tabletype(limitstype(8), funcref()));
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        my $func-idx = $emitter.function($function);
        $expression.i32-const(0);
        $expression.ref-func($func-idx);
        $expression.table-set($table-idx);
        $expression.i32-const(0);
        $expression.table-get($table-idx);
        $expression.ref-is-null();
        $emitter.export-function('test', $func-idx);
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], 0;
    }

    subtest 'table.size and table.grow' => {
        my $emitter = Wasm::Emitter.new;
        my $table-idx = $emitter.table(tabletype(limitstype(8), externref()));
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        $expression.table-size($table-idx);
        $expression.ref-null(externref());
        $expression.i32-const(4);
        $expression.table-grow($table-idx);
        $expression.drop();
        $expression.table-size($table-idx);
        $expression.i32-add();
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], 8 + 12;
    }

    subtest 'table.fill and table.copy' => {
        my $emitter = Wasm::Emitter.new;
        my $table-idx-a = $emitter.table(tabletype(limitstype(8), funcref()));
        my $table-idx-b = $emitter.table(tabletype(limitstype(8), funcref()));
        my $type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:$type-index, :$expression);
        my $func-idx = $emitter.function($function);
        $expression.i32-const(0);
        $expression.ref-func($func-idx);
        $expression.i32-const(4);
        $expression.table-fill($table-idx-a);
        $expression.i32-const(2);
        $expression.i32-const(0);
        $expression.i32-const(4);
        $expression.table-copy($table-idx-b, $table-idx-a);
        $expression.i32-const(5);
        $expression.table-get($table-idx-b);
        $expression.ref-is-null();
        $emitter.export-function('test', $func-idx);
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [], 0;
    }

    subtest 'call_indirect, table constructed from functions in globals' => {
        my $emitter = Wasm::Emitter.new;
        my $table-idx = $emitter.table(tabletype(limitstype(4), funcref()));
        my $callee-type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $caller-type-index = $emitter.function-type: functype(resulttype(i32()), resulttype(i32()));
        # Declare four functions returning integers. We also have to put them
        # into some kind of declaration to make them possible to reference;
        # use a global for that.
        my @funcs = do for ^4 {
            my $expression = Wasm::Emitter::Expression.new;
            my $function = Wasm::Emitter::Function.new(:type-index($callee-type-index), :$expression);
            $expression.i32-const(4 * $_);
            my $func-index = $emitter.function($function);
            my $ref-expression = Wasm::Emitter::Expression.new;
            $ref-expression.ref-func($func-index);
            $emitter.global(globaltype(funcref()), $ref-expression);
            $func-index
        }
        # Put them into a table, then use it for call_indirect testing.
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:type-index($caller-type-index), :1parameters, :$expression);
        for ^4 {
            $expression.i32-const($_);
            $expression.ref-func(@funcs[$_]);
            $expression.table-set($table-idx);
        }
        $expression.local-get(0);
        $expression.call-indirect($callee-type-index, $table-idx);
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [0], 0;
        is-function-output $buf, [1], 4;
        is-function-output $buf, [2], 8;
        is-function-output $buf, [3], 12;
    }

    subtest 'call_indirect, table constructed from functions in declarative elements' => {
        my $emitter = Wasm::Emitter.new;
        my $table-idx = $emitter.table(tabletype(limitstype(4), funcref()));
        my $callee-type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $caller-type-index = $emitter.function-type: functype(resulttype(i32()), resulttype(i32()));
        # Declare four functions returning integers, using declarative
        # elements to declare them up-front.
        my @indices;
        my @ref-exprs;
        for ^4 {
            my $expression = Wasm::Emitter::Expression.new;
            my $function = Wasm::Emitter::Function.new(:type-index($callee-type-index), :$expression);
            $expression.i32-const(4 * $_);
            my $func-index = $emitter.function($function);
            my $ref-expression = Wasm::Emitter::Expression.new;
            $ref-expression.ref-func($func-index);
            @indices.push($func-index);
            @ref-exprs.push($ref-expression);
        }
        $emitter.elements(Wasm::Emitter::Elements::Declarative.new(:type(funcref()), :init(@ref-exprs)));
        # Put them into a table, then use it for call_indirect testing.
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:type-index($caller-type-index), :1parameters, :$expression);
        for ^4 {
            $expression.i32-const($_);
            $expression.ref-func(@indices[$_]);
            $expression.table-set($table-idx);
        }
        $expression.local-get(0);
        $expression.call-indirect($callee-type-index, $table-idx);
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [0], 0;
        is-function-output $buf, [1], 4;
        is-function-output $buf, [2], 8;
        is-function-output $buf, [3], 12;
    }

    subtest 'call_indirect, table populated by active elements' => {
        my $emitter = Wasm::Emitter.new;
        my $table-idx = $emitter.table(tabletype(limitstype(4), funcref()));
        my $callee-type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $caller-type-index = $emitter.function-type: functype(resulttype(i32()), resulttype(i32()));
        # Declare four functions returning integers, and add an active
        # elements section.
        my @indices;
        my @ref-exprs;
        for ^4 {
            my $expression = Wasm::Emitter::Expression.new;
            my $function = Wasm::Emitter::Function.new(:type-index($callee-type-index), :$expression);
            $expression.i32-const(4 * $_);
            my $func-index = $emitter.function($function);
            my $ref-expression = Wasm::Emitter::Expression.new;
            $ref-expression.ref-func($func-index);
            @indices.push($func-index);
            @ref-exprs.push($ref-expression);
        }
        my $offset = Wasm::Emitter::Expression.new;
        $offset.i32-const(0);
        $emitter.elements: Wasm::Emitter::Elements::Active.new:
                :type(funcref()), :init(@ref-exprs),
                :table-index($table-idx), :$offset;
        # Use table use it for call_indirect testing.
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:type-index($caller-type-index), :1parameters, :$expression);
        $expression.local-get(0);
        $expression.call-indirect($callee-type-index, $table-idx);
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [0], 0;
        is-function-output $buf, [1], 4;
        is-function-output $buf, [2], 8;
        is-function-output $buf, [3], 12;
    }

    subtest 'call_indirect, table constructed from functions loaded from passive elements' => {
        my $emitter = Wasm::Emitter.new;
        my $table-idx = $emitter.table(tabletype(limitstype(4), funcref()));
        my $callee-type-index = $emitter.function-type: functype(resulttype(), resulttype(i32()));
        my $caller-type-index = $emitter.function-type: functype(resulttype(i32()), resulttype(i32()));
        # Declare four functions returning integers, using declarative
        # elements to declare them up-front.
        my @indices;
        my @ref-exprs;
        for ^4 {
            my $expression = Wasm::Emitter::Expression.new;
            my $function = Wasm::Emitter::Function.new(:type-index($callee-type-index), :$expression);
            $expression.i32-const(4 * $_);
            my $func-index = $emitter.function($function);
            my $ref-expression = Wasm::Emitter::Expression.new;
            $ref-expression.ref-func($func-index);
            @indices.push($func-index);
            @ref-exprs.push($ref-expression);
        }
        my $elements-idx = $emitter.elements: Wasm::Emitter::Elements::Passive.new:
                :type(funcref()), :init(@ref-exprs);
        # Load elements into a table, then use it for call_indirect testing.
        my $expression = Wasm::Emitter::Expression.new;
        my $function = Wasm::Emitter::Function.new(:type-index($caller-type-index), :1parameters, :$expression);
        $expression.i32-const(0);
        $expression.i32-const(0);
        $expression.i32-const(4);
        $expression.table-init($elements-idx, $table-idx);
        $expression.elem-drop($elements-idx);
        $expression.local-get(0);
        $expression.call-indirect($callee-type-index, $table-idx);
        $emitter.export-function('test', $emitter.function($function));
        my $buf = $emitter.assemble();
        pass 'Assembled module';
        is-function-output $buf, [0], 0;
        is-function-output $buf, [1], 4;
        is-function-output $buf, [2], 8;
        is-function-output $buf, [3], 12;
    }
}
else {
    skip 'No wasmtime available to run test output; skipping';
}

done-testing;
