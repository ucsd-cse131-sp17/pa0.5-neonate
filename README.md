# A First Compiler – Neonate

Today we're going to implement a compiler.  It will be called Neonate, because
it's fun to name things and the name will fit a theme in future weeks.

It's not going to be terrifically useful, as it will only compile a very small
language – integers.  That is, it will take a user program (a number), and
create an executable binary that prints the number.  There are no files in this
repository because the point of the assignment is for you to see how this is
built up from scratch.  That way, you'll understand the infrastructure that
future assignments' support code will use.

## The Big Picture

The heart of each compiler we write will be an OCaml program that takes an
input program and generates assembly code.  That leaves open a few questions:

- How will the input program be handed to, and represented in, OCaml?
- How will the generated assembly code be run?

Our answer to the first question is going to be simple for today: we'll expect
that all programs are files containing a single integer, so there's little
“front-end” for the compiler to consider.  Most of this assignment is about
the second question – how we take our generated assembly and meaningfully run
it while avoiding both (a) the feeling that there's too much magic going on,
and (b) getting bogged down in system-level details that don't enlighten us
about compilers.

## The Wrapper

(The idea here is directly taken from [Abdulaziz
Ghuloum](http://scheme2006.cs.uchicago.edu/11-ghuloum.pdf)).

Our model for the code we generate is that it will start from a C-style
function call.  This allows us to do a few things:

- We can use a C program as the wrapper around our code, which makes it
  somewhat more cross-platform than it would be otherwise
- We can defer some details to our C wrapper that we want to skip or leave
  until later

So, our wrapper will be a C program with a traditional main that calls a
function that we will define with our generated code:

```
#include <stdio.h>

extern int our_code_starts_here() asm("our_code_starts_here");

int main(int argc, char** argv) {
  int result = our_code_starts_here();
  printf("%d\n", result);
  return 0;
}
```

So right now, our compiled program had better return an integer, and our
wrapper will handle printing it out for us.  The syntax
`asm("our_code_starts_here")` tells a compiler like `gcc` or `clang` to not do
any platform-specific name-alterations, and to use the provided name exactly
as it appears.  This makes it so the names that the compiler tries to find in
object files don't vary across platforms (not something I'd recommended in
general, but quite useful for our purposes).

We can put this in a file called `main.c`.  If we try to compile it, we get an
error:

```
⤇ clang -g -o main main.c
Undefined symbols for architecture x86_64:
  "our_code_starts_here", referenced from:
      _main in main-1a486d.o
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

That's because it's our job to define `our_code_starts_here`.  That's what
we'll do next.

## Hello, x86

Our next goal is to:

- Write an assembly program that defines `our_code_starts_here`
- Link that program with `main.c` and create an executable

In order to write assembly, we need to pick a syntax and an instruction set.
We're going to generate 32-bit x86 assembly, and use the so-called Intel
syntax (there's also an AT&T syntax, for those curious), because [I like a
particular guide](http://www.cs.virginia.edu/~evans/cs216/guides/x86.html)
that uses the Intel syntax, and because it works with the particular assembler
we'll use.

Here's a very simple assembly program, matching the above constraints, that
will act like a C function of no arguments and return a constant number (`37`)
as the return value:

```
section .text
global our_code_starts_here
our_code_starts_here:
  mov eax, 37
  ret
```

The pieces mean, line by line:

- `section .text` – Here comes some code, in text form!
- `global our_code_starts_here` – This assembly code defines a
  globally-accessible symbol called `our_code_starts_here`.  This is what
  makes it so that when we generate an object file later, the linker will know
  what names come from where.
- `our_code_starts_here:` – Here's where the code for this symbol starts.  If
  other code jumps to `our_code_starts_here`, this is where it begins.
- `mov eax, 37` – Take the constant number 37 and put it in the register
  called `eax`.  This register is the one that compiled C programs expect to
  find return values in, so we should put our “answer” there.
- `ret` – Do mechanics related to managing the stack which we will talk about
  in much more detail later, then jump to wherever the caller of
  `our_code_starts_here` left off.

We can put this in a file called `our_code.s` (`.s` is a typical extension for
assembly code), and then we just need to know how to assemble and link it with
the main we wrote.

## Hello, `nasm`

We will be using a program called [nasm](http://www.nasm.us/) as our
assembler, because it works well across a few platforms, and is simple to use.
The main way we will use it is to take assembly (`.s`) files and turn them
into object (`.o`) files that traditional compilers like `clang` or `gcc` can
work with.  The command we'll use to build with nasm is:

```
⤇ nasm -f elf32 -o our_code.o our_code.s
```

This creates a file called `our_code.o` in [Executable and Linkable
Format](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format).  We
won't go into detail (yet, depending on what we have time for in the course)
about this binary structure.  For our purposes, it's simply a version of the
assembly we wrote that our particular operating system understands.

If you are on OSX, you can use `-f macho` rather than `-f elf32`, which will
produce an OSX-compatible object file.

With this in hand, and we ought to be able to compile it into a binary along
with a C source file just like any other object file generated from C.  For
example:

```
⤇ clang -g -o our_code main.c our_code.o
```

But this gives an error:

```
/usr/bin/ld: i386 architecture of input file `our_code.o' is incompatible with i386:x86-64 output
collect2: error: ld returned 1 exit status
```

This happens because the default (on the department machines) is to generate
binaries for 64-bit x86, and we're targeting 32-bit x86.  So we need to tell
`clang` that's what we want:

```
⤇ clang -g -m32 -o our_code main.c our_code.o
```

Now we can run our code:

```
⤇ ./our_code
37
```

Note that I will almost always include the `-g` option on uses of `clang`,
because it's always handy to have debugging information turned on.

## Hello, Compiler

With this pipeline in place, the only step left is to write an OCaml program
that can generate assembly programs.  Then we can automate the process and get
a pipeline from user program all the way to executable.

A very simple compiler might just take the name of a file, and output the
compiled assembly code on standard output.  Let's try that; here's a simple
`compiler.ml` that takes a file as a command line argument, expects it to
contain a single integer on one line, and generates the corresponding assembly
code:

```
open Printf

(* A very sophisticated compiler - insert the given integer into the mov
instruction at the correct place *)
let compile (program : int) : string =
  sprintf "
section .text
global our_code_starts_here
our_code_starts_here:
  mov eax, %d
  ret\n" program;;

(* Some OCaml boilerplate for reading files and command-line arguments *)
let () =
  let input_file = (open_in (Sys.argv.(1))) in
  let input_program = int_of_string (input_line input_file) in
  let program = (compile input_program) in
  printf "%s\n" program;;
```

Put this into `compiler.ml`, and create another file `87.int` that
contains just the number 87, then run:

```
⤇ ocaml compiler.ml 87.int

section .text
global our_code_starts_here
our_code_starts_here:
  mov eax, 87
  ret
```

How exciting!  We can redirect the output to a file, and get an entire
pipeline of compilation to work out:


```
⤇ ocaml compiler.ml 87.int > 87.s
⤇ nasm -f elf32 -o 87.o 87.s
⤇ clang -m32 -o 87.run main.c 87.o
⤇ ./87.run
87
```

If we like, we could capture this set of dependencies with a `make` rule:

```
%.run: %.o
	clang -m32 -o $@ main.c $<

%.o: %.s
	nasm -f elf32 -o $@ $<

%.s: %.int
	ocaml compiler.ml $< > $@
```

If we put that in a `Makefile`, then we can just run:

```
⤇ make 87.run
```

and we have the definition of our compiler.

## Is that it?

Of course, this “just” a bunch of boilerplate.  It got us to the point
where we have an OCaml program that's defining our translation from input
program to assembly code.  Our input programs are pretty boring, so those will
need to get more sophisticated, and correspondingly the function `compile`
will need to become more impressive.  That's where our focus will be in the
coming weeks.

In the meantime, you have a little compiler to play with.  Can you think of any
other interesting input program formats to try, or tweaks to the generated
output to play with?  For example, could you make it take in a list of numbers,
and generate instructions that add them all up?

## Handin

Commit and hand in your working `compile.ml`, `main.c`, and a binary file
`87.run` generated by using the instructions above.

Also, make sure to fill out this form so we can correctly associate your Github
username with your UCSD account:

https://goo.gl/forms/KGPjV1L6HtYXmo4H3

