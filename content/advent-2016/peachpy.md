+++
linktitle = "PeachPy"
date = "2016-12-02T00:00:00"
author = [ "Damian Gryski" ]
title = "Writing Go assembly functions with PeachPy"
series = ["Advent 2016"]
+++

**What is PeachPy**

[PeachPy](https://github.com/Maratyszcza/PeachPy) is a Python-based framework
for writing modules in assembly.  It automates away some of the details and
allows you to use Python to generate repetitive assembly code sequences.

PeachPy supports writing modules that you can use directly from Go for x86-64.
(It also supports NaCl and syso modules, but I won't cover those in this post.)

This post is going to be mostly about what you need to know about integrating
PeachPy and less a tutorial about PeachPy specifically.

All the code for this post is at
[github.com/dgryski/peachpy-examples](https://github.com/dgryski/peachpy-examples).

**A Simple Example**

Lets start with a simple function: one that takes two integers and returns their sum.

[embedmd]:# (args/add.py)
```py
import peachpy.x86_64

f1 = Argument(uint64_t)
f2 = Argument(uint16_t)

with Function("add", (f1, f2), uint64_t) as function:
    reg_f1 = GeneralPurposeRegister64()
    reg_f2 = GeneralPurposeRegister64()

    LOAD.ARGUMENT(reg_f1, f1)
    LOAD.ARGUMENT(reg_f2, f2)

    ADD(reg_f1, reg_f2)

    RETURN(reg_f1)
```

Next, we'll add a function stub with a `go:generate` directive:

[embedmd]:# (args/add_stub.go)
```go
// +build amd64

package main

//go:generate python -m peachpy.x86_64 add.py -S -o add_amd64.s -mabi=goasm
func add(a uint64, b uint16) uint64
```

When we run `go generate`, we get the following assembly code:

[embedmd]:# (args/add_amd64.s asm)
```asm
// Generated by PeachPy 0.2.0 from add.py


// func add(f1 uint64, f2 uint16) uint64
TEXT ·add(SB),4,$0-24
	MOVQ f1+0(FP), AX
	MOVWQZX f2+8(FP), BX
	ADDQ BX, AX
	MOVQ AX, ret+16(FP)
	RET
```

Let's go back through the PeachPy code piece-by-piece.

[embedmd]:# (args/add.py /f1/ /(?m:^$)/)
```py
f1 = Argument(uint64_t)
f2 = Argument(uint16_t)
```

First, we declare our two arguments.  The types we declare here are used to
determine where and how the arguments are passed to the function, and for the
auto-generated function prototype.  The Go the function prototype is only in a
comment in the generated assembly file.  There is no capability to match these
arguments to the actual function stub, so a mismatch here won't be detected.

[embedmd]:# (args/add.py /with.*/)
```py
with Function("add", (f1, f2), uint64_t) as function:
```

This line declares a function `add` with two arguments `(f1, f2)` and with a
return value of type `uint64_t`.

[embedmd]:# (args/add.py /.*reg_f1/ /(?m:^$)/)
```py
    reg_f1 = GeneralPurposeRegister64()
    reg_f2 = GeneralPurposeRegister64()
```

Next, we acquire two registers.  In this case, we don't care what the actual
registers are. PeachPy will assign two for us.

[embedmd]:# (args/add.py /.*LOAD.*reg_f1/ /(?m:^$)/)
```py
    LOAD.ARGUMENT(reg_f1, f1)
    LOAD.ARGUMENT(reg_f2, f2)
```

These are PeachPy pseudo-instructions which will load the arguments into the
chosen registers.  This is where details are hidden allowing portable assembly.
With the Go calling convention the arguments are passed on the stack, and so
PeachPy generates the appropriate `MOV` instructions.  If PeachPy generates
code for the regular C calling convention, those will load the arguments from
the appropriate registers instead.

[embedmd]:# (args/add.py /.*ADD.*/ /RETURN.*/ )
```py
    ADD(reg_f1, reg_f2)

    RETURN(reg_f1)
```

This last block does the addition and then sets the return value.  Here again
`RETURN` is a PeachPy pseudo-instruction which will generate the appropriate
store into the stack for Go's calling convention.

Note also that the operands are Intel syntax order (destination first), rather
than the Plan9/AT&T order (source first).  PeachPy generates the correct
assembly syntax depending on the target.

We can then call this as a normal Go function.

[embedmd]:# (args/main.go /func main/ /}/ )
```go
func main() {
	sum := add(100, 20)
	fmt.Printf("sum = %+v\n", sum)
}
```

**Working with structs**

This example will talk about passing a struct by reference to a function.

First, in order to be able to access any of the fields, you need to know where
their offset relative to the base address of the struct.  The `unsafe.Offsetof`
function will do this for you, but so can Dominik Honnef's `structlayout` tool.

For example, given

[embedmd]:# (struct/main.go /type foo struct/ /}/)
```go
type foo struct {
	zot uint16
	bar uint16
	qux uint64
}
```

We can query the offsets of all the fields with:

```
    $ structlayout github.com/dgryski/peachpy-examples/struct foo
    foo.zot uint16: 0-2 (size 2, align 2)
    foo.bar uint16: 2-4 (size 2, align 2)
    padding: 4-8 (size 4, align 0)
    foo.qux uint64: 8-16 (size 8, align 8)
```

And here's our PeachPy code:

[embedmd]:# (struct/add.py)
```py
import peachpy.x86_64

f = Argument(ptr())

with Function("add", (f,), uint64_t) as function:
    reg_f_base = GeneralPurposeRegister64()

    LOAD.ARGUMENT(reg_f_base, f)

    v = GeneralPurposeRegister64()

    # move and zero-extend 16-bit value at addr reg_f_base+2
    MOVZX(v, word[reg_f_base+2])
    ADD(v, [reg_f_base+8])

    RETURN(v)
```

This time we have a few differences.  First, the argument type is declared as a
`ptr()`.  This is the equivalent of a C's `void *` Go's `uintptr`.

Second, when declaring the function itself, there is only a single argument
which, as PeachPy is expecting a tuple as the second argument, we have to write
as `(f, )`.

The body of the function starts off the same.  We load the address of the
struct (the first argument) into `reg_f_base`.

Next, we declare a temporary register `v` to store the sum.

Our next two instructions load the value of `bar` (a word-sized value at offset
2) and add `qux` (offset 8)

Finally, we set `v` as the return value from the function.

[embedmd]:# (struct/add_stub.go /..go:generate/ $)
```go
//go:generate python -m peachpy.x86_64 add.py -S -o add_amd64.s -mabi=goasm
//go:noescape
func add(f *foo) uint64
```

Our stub file includes an additional directive before the function declaration.
The `//go:noescape` directive tells the compiler that the pointer passed to
this function does not escape to the heap or into the return values.  Without
this, the compiler would have no choice but to allocate the struct on the heap.
Now, the compiler's escape analysis can tell it to potentially be allocated on
the stack instead (as it will be with our `main` function).  This can be a
significant win.  On the other hand, stack allocation can break the alignment
needed by certain SSE instructions, requiring the use of the unaligned version
instead.

Here is our `main`:

[embedmd]:# (struct/main.go /func main/ /(?m:^})/)
```go
func main() {
	var f = foo{bar: 200, qux: 50000}
	sum := add(&f)
	fmt.Printf("sum = %+v\n", sum)
}
```

And running it we get the expected result:

```
sum = 50200
```

**Working with slices**

There are two ways to work with slices.

You can pass the pass the address of the first element and work with that but
then your pure-Go version and your asm version will have different parameter
lists.  This matches more closely with how C would call your code and makes it
easier to use your assembly code for both C and Go.

On the other hand, since you can't do pointer-arithmetic in Go, your pure-Go
version (you *are* writing pure-Go versions of your assembly routines, right?)
will still need to be passed a slice.  Which means you're going to have code that looks like

```
var total uint64
if useGo {
   total = addGo(x)
} else {
   total = addAsm(&x[0], len(x))
}
```

But if you're just writing Go code, it's nicer if they're the same.  (You can
still call your slice-expecting-assembly routine from C -- you just need to
pass the length twice to pretend to be the capacity).

Lets look at an example.  Here are the arguments to the function:

[embedmd]:# (slice/add.py /s_base/ /(?m:^$)/)
```py
s_base = Argument(ptr())
s_len = Argument(size_t)
s_cap = Argument(size_t)
```

A slice is passed as three arguments: a pointer to the data, the length, and
the capacity.  Technically, `len` and `cap` are signed ints.  However, we're
going to use `size_t` instead, even though it's unsigned.  (The signedness here
is only used for cosmetic purposes when generating the prototype comment for
the generated assembly and doesn't affect the instructions at all.  Also, using
`size_t` is semantically nicer than using `ptrdiff_t` which is an appropriately
sized signed integer.  PeachPy only uses the size of the integer when
generating stack offsets.  If you want to be exact and you know you're only
going to be running on amd64, you could use the exact `int64_t` instead.

Here our function declaration mentions the three arguments we're expecting.  We
can also see the use of PeachPy's `with Loop()` construct to sum up all the
elements of the slice.

[embedmd]:# (slice/add.py /with Function.*/ /RETURN.*/)
```py
with Function("add", (s_base,s_len,s_cap), uint64_t) as function:
    reg_s_base = GeneralPurposeRegister64()
    reg_s_len = GeneralPurposeRegister64()

    LOAD.ARGUMENT(reg_s_base, s_base)
    LOAD.ARGUMENT(reg_s_len, s_len)

    total = GeneralPurposeRegister64()
    XOR(total, total)

    with Loop() as loop:
        ADD(total, [reg_s_base])
        ADD(reg_s_base, 8)
        SUB(reg_s_len, 1)
        JNZ(loop.begin)

    RETURN(total)
```

While our assembly stub just lists a single slice:

[embedmd]:# (slice/add_stub.go /func.*/)
```go
func add(s []uint64) uint64
```

The `go vet` tool will check that the assembly arguments for a slice are named
correctly.  If you have an argument `s` which is a slice, the three arguments
to your assembly function should be `s_base`, `s_len`, and `s_cap`.

**Working with strings**

A string is just a slice without a capacity, so you only have to declare
two arguments: a data pointer and a length.

**Debugging**

Delve supports single-stepping through assembly code, which is nice.  It
doesn't yet support [viewing any of the extended
registers](https://github.com/derekparker/delve/issues/666).

Since PeachPy supports the standard C calling convention, you can also call
your assembly routines from C and debug it with any of the standard Linux
debuggers without having to worry about the Go runtime specifics.

For example, the slice code above we can generate a sysv elf object file with

[embedmd]:# (slice/c/gen.sh /python.*/)
```sh
python -m peachpy.x86_64 ../add.py -emit-c-header add.h -mimage-format=elf -o add_amd64.o -mabi=sysv
```

and call it from C with

[embedmd]:# (slice/c/add.c /int main/ /\n}/)
```c
int main() {
    uint64_t t[]={100, 200, 1000, 50000};
    printf("sum=%ld\n", add(&t, 4, 4));
}
```

**Testing**

Testing assembly code is no different from testing regular code.  However, one
suggestion is to always have a pure-Go version of whatever your assembly
routine is doing.  This can be used on platforms, but also as a base-version to
compare your optimized implementation with.  If you use build-tags to choose an
implementation at build-time, you'll probably need to rename the functions when
fuzzing so you can have both functions in your test binary.

The inputs can be from your regular test suite, or explored via go-fuzz. For
more on using go-fuzz for comparing implementations, please see [DNS Parser,
meet Go fuzzer](https://blog.cloudflare.com/dns-parser-meet-go-fuzzer/)

Finally, you can also leverage PeachPy and do your testing in Python.
This will require constructing argument lists via Python's `ctypes` module.

A simple example to test our code which works with slices:

[embedmd]:# (slice/add.py /if __name__/ $)
```py
if __name__ == "__main__":
    import ctypes
    add_asm = function.finalize(abi.detect()).encode().load()
    inp = [10,500,2000,50000]
    arr = (ctypes.c_ulonglong * len(inp))(*inp)
    g = add_asm(arr,len(arr),len(arr))
    assert(g == 52510)
```

By having your tests in your python code, they will be run each time PeachPy
generates the assembly code.

**Resources**

The official docs for the Go assembler at https://golang.org/doc/asm.  They're
useful to read, but remember that PeachPy will be taking care of the many of
the details for you regarding the syntax and calling convention.

The [PeachPy sources](https://github.com/Maratyszcza/PeachPy)

Finally, at GolangUK 2016, Michael Munday gave a talk on [Dropping Down: Go Functions in
Assembly](https://www.youtube.com/watch?v=9jpnFmJr2PE).
