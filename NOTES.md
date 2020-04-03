## CSCI 221, Spring 2020
# Lecture 08-2: memory access in MIPS

---

## Overview

In this lecture we'll see how to write (and call) functions within
MIPS assembly. This has us look more carefully at the `JAL` and `JR`
instructions, at the conventional roles of each of the MIPS registers,
how to maintain a call stack and each function's frame, and more
generally the function *calling conventions* that all MIPS32
programmers have agreed to follow. 

There is support in the processor hardware for calling functions,
namely, the ability to save the address following an executing
instruction into a register. This mechanism is key for function code
(the "callee") to return program control back to the part of the
program that called the function (the "caller's" *call site*).

However, most of the other things we'll learn today about function
calling in MIPS are *purely conventions*. There's no reason that
things couldn't be done differently, it's just that these rules 
are considered good and people have all agreed to adopt them.

Such is engineering practice.

By the way, check out Doug Ellard's notes on MIPS programming 
[on his personal site](http://ellard.org/dan/www/Courses/)
which include a chapter on these calling conventions.

---

## Outline

    0. Slack channel; office hour poll
    I. Recall how function calls work in C/C++
     A. example C++ functions and function calls
     B. some terminology: *caller*, *callee*, *call site* 
     C. stack picture; stack frame push/pop with function call/return
     D. Goal: write the same code in MIPS
    II. Digression on bit shifting operations
     A. multiplying by 10 example
     B. SLL and SRA instructions (there's also a MULT)
    III. Simple MIPS functions (no register or stack conventions)
     A. "leaf" calls: main->{two_digit,times100}
     B. use of ra, a0 and a1, v0
     C. version where two_digit "clobbers" t0 of main
     D. reasons for the stack and conventions
    IV. MIPS calling conventions and stack discipline
     A. managing the stack: fp and sp  
     B. example: four_digits
     C. convention for use of s0-s7 (callee-saved)
     D. convention for use of t0-t9 (caller-saved), a0-a3
     E. summary

---

## Logistics 

1. I've created a Slack channel named "csci221-s20-discuss"
on a "Reed Computer Science" Slack group. I've invited you all to
join that group, if you haven't already for another course, and
added you to that channel's list of partcipants. The TAs are 
there as well. Feel free to use that to ask us questions.

2. I've emailed you all a "WhenIsGood" poll so that I can figure
out good times to schedule office hours for both myself and the
TAs. Please take that poll!

---

## Recall: functions in C++

### Two examples

Here is some example code:

    int two_digits(int tens, int ones) {
        return 10 * tens + ones;
    }
    
    int times100(int number) {
        return 100 * number;
    }
    
    int main(void) {
        int A, B, C, D;
        cin >> A;
        cin >> B;
        cin >> C;
        cin >> D;
        int hi = two_digits(A,B);
        int lo = two_digits(C,D);
        int n = times100(hi) + lo;
        cout << n << endl;
    }

In the above, `main` calls `two_digits` twice. There are two
*call sites* within `main` where `two_digits` gets invoked.
Also, `main` calls `times100` at a single call site.

The code, when given single digit integers, say, 5, 1, 3, 7,
outputs the four-digit number with those as its digits,
5137, say.

    int two_digits(int tens, int ones) {
        return 10 * tens + ones;
    }
    
    int times100(int number) {
        return 100 * number;
    }
    
    int four_digits(int w, int x, int y, int z) {
        int hi = two_digits(w,x);
        int lo = two_digits(y,z);
        return times100(hi) + lo;
    }
    
    int main(void) {
        int A, B, C, D;
        cin >> A;
        cin >> B;
        cin >> C;
        cin >> D;
        int n = four_digits(A,B,C,D);
        cout << n << endl;
    }

In the above, `four_digits` takes over the role
of `main`. It calls `two_digits` and `times100`.
It is called my `main`.  

### Terminology

When talking about `main` calling `four_digits`,
we'll say that `main` is the "caller" and 
`four_digits` is the "callee." When talking about
`four_digits` calling `two_digits`, then `four_digits`
is the caller, and `two_digits` is the callee.

Thinking about the caller/callee relationship is
something that is critical in the construction
of software. We might say that `two_digits` is
providing a "useful service" to `four_digits`.
From that view, `four_digits` is a client
of the `two_digits` code.

Taking this further, the code for both functions
"enter into a software contract" because of those
two call sites, those two uses, of `two_digits`.
A `two_digits` client is expected to send two integer
values (each a value between 0 and 9). And it
expects to get a certain integer back.

### Recall: the call stack; stack frames

Recall that we've talked a bit about the stack mechanism, the memory
component that is used to store the local variables of functions when
compiled C++ executes.  We named this the *call stack*. It's an
invented construct, but it is something that our C++ code maintains
when it executes.

The call stack mechanism, for the most part, is hidden from us. The
compiler generates low-level code to manage it.  Nevertheless, the
compiler does it on our behalf. And, because C++ (I feel) requires a
bit of understanding of the hidden low-level details, it seems
important to learn how it all works.

Any time a function is called, the low-level code that executes
creates a stack frame to hold the contents of its local variables. The
size of that stack frame (almost always) is known ahead of time by the
compiler---it is some number of bytes---and that frame's space is
"pushed" onto the top of the call stack. It is the newest frame, and
the chain of calls that led to this call, their frames are sitting on
the stack as well.

Recall, then, that when `main` calls `four_digits`, which then calls
`two_digits`, those three calls each have a frame on the stack. The
stack lives in memory and grows downward with each subsequent
call. The deeper the number of calls have stacked up, then all those
frames are "live" or "active", with each function waiting for the
return of its callee so that it can continue executing.

When a function returns, its stack frame (in essence) gets popped off
the call stack. And program control returns to the line after its call
site in its caller.

Today we make explicit how all this actually occurs in MIPS
assembly. We'll write code that mimics the activity of the C++ code
above and also manages the call stack. We'll also see how parameters
are communicated from the caller to the callee, and how the return
value is communicated back to the caller from the callee.

---

## Shifting bits

Before we begin this investigations, I want to take a brief digression
on how multiplication works at a low level in instruction sets like
MIPS's.  Our first MIPS assignment had us multiply using repeated
addition, but that is *definitely* how products are computed in 
modern processors.  

Instead, MIPS32 has a `MULT` instruction that takes two register
sources and then places their product into special registers. Since
the product of two 32-bit integers, in general, requires 64-bits
to be expressed, the MIPS architecture has two special 32-bit
registers, designated `hi` and `lo` for storing the high and 
low order bits of the product it computes.

Rather than focus on that instruction, I instead want to briefly
touch on how multiplication can be done by "bit shifting" rather
than by a special instruction. I'll illustrate this only with
two examples, namely `two_digits` and `times100`. (These can
be found as `.asm` files in the lecture's samples.) How can
we multiply by 10? How can we multiply by 100?

Recall that the binary representation of 10 is `1010`. That is
10 = 8 + 2.  So that means that 10x is the same as 8x + 2x.
How is that useful? Recall that multiplying by powers of 2 is
just the result of shifting the bits of a number's binary
representation some number of positions. 

Consider 7. It's binary representation is `111`. And so 14
is `1110`. And 56 os `111000`.  In the former, we shifted
7's bits one place left. In the latter, we shifted 7's
bits three places to the left. 

### Shifting left

In MIPS we can use the instruction

&nbsp;&nbsp;&nbsp;&nbsp;`SLL` *destination*`,`*source*`,`*number-of-places*

This sets the register *destination* equal to the value of *source*, but shifted
that number of places to the left. So, in the following code

    sll $t1,$t0,1
    sll $t2,$t0,3

if `t0` held 7, then `t1` would hold 14 and `t3` would hold 56. That means that
the code

    sll $t1,$t0,1
    sll $t2,$t0,3
    addu $t0,$t1,$t2

has the effect of multiplying `t0` by 10.

How about 100? It's binary representation is `1100100`. And so the code

    sll $t1,$t0,2
    sll $t2,$t0,5
    sll $t3,$t0,6
    addu $t0,$t1,$t2
    addu $t0,$t0,$t3

**Exercise**: can you do this using only one other register than `t0`?

### Shifting right is division by two

Recall also that dividing by powers of two is easy in binary.
100 divided by four is written as `11001` in binary. And the
quotient of 100 divided by 16 is 6 (ignoring the remainder)
and its binary code is just `110`. This is just `1100100` 
shifted right three times, tossing out any 1s that "fall off
the left." There are two shift right instructions in MIPS:

&nbsp;&nbsp;&nbsp;&nbsp;`SRL` *destination*`,`*source*`,`*number-of-places*  
&nbsp;&nbsp;&nbsp;&nbsp;`SRA` *destination*`,`*source*`,`*number-of-places*  

The first is "shift right logical." It's the analogue to `SLL` which is
"shift left logical."  The word "logical" here means that a bit of 0 is
shifted in, in both cases. When `111111..11` (32 bits of 1) is shifted 
left "logically," the result is `111111..10` (31 bits of 1 followed by 0).
When shifted right logically the result is `011111..11` (0 followd by 31
bits of 1). 

Recall that the leftmost bit, in two's complement encoding, is the "sign
bit".  The `111111..11` represents -1. Its leftmost bit is 1. `011111..11`
is the largest positive number that fits in 32 bits using two's complement.
Its leftmost bit is 0.

`SRA` means "shift right arithmetically." The difference is that, when
the bits are shifted right, rather than shifting a 0 bit in on the left,
the sign bit is repeated. The `SRA` of `111111..11`, then, is 
`111111..11`. For any other negative number other than -1, the `SRA`
gives that number divided by two. Same with any positive integer.
For -1, the result is weird.

---

##  Leaf call


### Client code

Let's look at a snippet of `main`, written in MIPS, that mimics the
code for our first C++ example:

    main:
    ... # put inputs A,B,C,D into s0-s3
        move $a0,$s0
        move $a1,$s1
        jal  two_digits
        move $s0,$v0

        move $a0,$s2
        move $a1,$s3
        jal  two_digits
	move $s1,$v0

        move $a0,$s0
        jal  times100
        addu $a0,$v0,$s1
    ... # output $a0  
        
Here are some observations you should make
about this:

* `JAL` instructions are MIPS function call sites.  The name stands
for "JUMP AND LINK." The effect of each is to jump to the code label
specified. That function's code is expected, when it's done computing,
to jump back, and to the line just below that `JAL`.  The phrase
"and link" hints that there is an underlying processor mechanism
that makes the "jump back," or *return*, happen.

* Arguments are passed in the "argument registers" `a0` and `a1`
for two-parameter functions. It is passed using `a0` in a one-parameter
function. There are four such registers `a0` through `a3`, and so up to
four parameters can be passed that way. (The others, when more than
four, are passed using the stack.)

* The return value is placed in the "return value" register `v0`
by the function.

Passing the parameters by setting the argument registers is the
contract that the caller fulfills. The called function knows where
to find those parameter values. 

Returning the value by setting that return value register is the contract
that the callee fulfills. The caller knows where to get that value, once
the called function returns.

### The function code

Here is the code for the two functions

    two_digits:
        sll  $t0,$a0,1
        sll  $t1,$a0,3
        addu $v0,$t1,$t0
        addu $v0,$v0,$a1
        jr   $ra

    times100:
        sll  $v0,$a0,2
        sll  $t0,$a0,5
        sll  $t1,$a0,6
        addu $v0,$t0,$v0
        addu $v0,$t1,$v0
        jr   $ra

In the first function, the value `10.a0 + a1` is computed and
the result is put in `v0`. In the second, `v0` is set to
`100.a0`. These rely on the convention of passing in
the argument registers and returning the result in `v0`.

And then the `jr $ra` instruction performs the actual 
return. `JR` stands for "JUMP to the address in a REGISTER"
and a register named `ra` is used.

This, I hope, tells you what `JAL` does. Before `JAL` jumps to the
labelled instruction of the function being called, that instruction
sets the register `ra` to the address of the instruction just
following the call site.  By doing this, it allows the callee to
return control back to the caller, and at the appropriate line of the
caller's code.

### Some questions

What if, instead of using registers `s0`-`s3`, `main` had chosen to
use registers `t0`-`t3`? Then the second call to `two_digits` would
have written over the result of the first call, stored in `t0`.
And alsi the call to `times100` would have clobbered the result of
the second call to `two_digits`, kept in `t1`.

We see the danger, then, in using registers to store the local 
variables of functions. Registers are actually global variables.
All parts of the program have access to those same 32 registers.
When writing a large piece of code, then, with a lot of functions,
using only registers, you have to somehow juggle them, and know
about how every part of the code acts and behaves in order to
have them not "step on each other toes." 

In addition, systems like C++ rely on *separate compilation*.
The compiler independently compiles one set of functions, and
those might get linked to some other code, also separately compiled.
In that case, the compiler can't know ahead of time how that
client code works when it compiles the code it will eventually call.

What makes the task impossible is that function calls can be made
arbitrarily deep. Code main, might call f, which might call g, etc.
With that many functions, and only so many registers, most code will
run out of registers, For example, recursive code, either with a
single function or mutually recursive functions, could need a large
number of local valraibles to be kept around. A fixed number of
registers can't be used in that case.

We know the answer to the last two challenges: we know that C++
manages a call stack. But, in addition, (and probably primarily
because of separate compilation) functions are written to
use the temporary registers `t0`-`t9` and `s0-`s7` in 
very disciplined ways.

---

## MIPS calling conventions

Let's start this section by accepting the fact that function code 
compiled to MIPS will choose to keep some of its local variables
on its stack frame. And let's also accept that the stack frame
size needed is known, ahead of time, for each function we write.
We'll assume that stack frames are of fixed size, always the
same size anytime some function `f` is called, though the size 
for function `f` may differ than the size for function `g`.

### The stack discipline
 
There are two registers in MIPS that point to the start and end
of a function's stack frame. The register `fp` points to the
byte just above the highest-addressed byte of the frame.
The register `sp` points to the lowest-addressed byte of the 
frame. Since the stack grows downward, then `fp` also points
to the lowest-address byte of its caller's frame.

When a function is first called it moves `sp` and `fp` to delineate
its frame just below its caller's frame. When a function returns
it restores `fp` and `sp` to their original locations so that
the caller can continue as it did before it was called, accessing
its frame as if nothing had changed. 

The callee bumps `sp` and `fp` lower at the start of its code,
and then bumps them back up at the end of its code, just before
executing `jr $ra` and returning control back to its caller.

Here, then maybe, is how we should begin the code for `four_digits`,
assuming its stack frame needs to be 32 bytes:

    four_digits:
        move $fp,$sp
        addiu $sp,$sp,-32
        ...

We're growing the stack downward, and laying out 32 bytes between
`fp` and `sp`. But then we've lost track of the caller's `fp`.
So we save it to the top of our frame first:

    four_digits:
        sw    $fp,-4($sp)
        move  $fp,$sp
        addiu $sp,$sp,-32
        ...

That saves it in the four bytes just below our caller's frame.
But note also that we are going to make calls using `JAL` to
`two_digits` and `times100`.  This means that we will need to
save the callers `ra` so that the two functions we call can
rely on our `ra` to return back to us. So we do this instead:

    four_digits:
        sw    $ra,-4($sp)
        sw    $fp,-8($sp)
        move  $fp,$sp
        addiu $sp,$sp,-32
        ...

This saves `fp` and `ra` so that we can restore them later
for our return to our caller. The code for `four_digits`, then,
has this code at the bottom:

        ...
        move  $a0,$s0
        jal   times100
        addu  $v0,$v0,$s1   # set the return value

        addiu $sp,$sp,32
        lw    $ra,-4($sp)
        lw    $fp,-8($sp)
        jr    $ra

These last four lines restores the caller's `sp` and `fp`, and loads
the proper return address into `ra` to jump back to the caller. Just
above those four lines, we set the return value register `v0`.

Since we are the callee with respect to the caller then, in MIPS,
`fp`, `sp`, and `ra` are dubbed *callee-saved registers*. By contract
with the caller, the callee is expected to makes sure their values
aren't clobbered by its activity. So it saves them at the start and
then restores them before its return.

### More conventions: other callee saved registers

It turns out that the code we just wrote is not following the proper
conventions for MIPS coding. The MIPS function calling conventions
require that the temporary registers `s0`-`s7` be preserved by the
callee also.  If our function plans to use them, then we must save
them somewhere before we do, and then restore them when we are done
using them, sometime before we return. They are also callee-saved
registers.

Our code can assume that`a0`-`a3` hold the four parameter values it's sent,
so it need not use `s0`-`s3` the way that our `main` above did. But let's
still assume that it wants to use `s0` for the result of the first call
to `two_digits` and `s1` for the result of the second call. And so
it must save these two on the stack, too:

    four_digits:
        sw    $ra,-4($sp)
        sw    $fp,-8($sp)
        sw    $s0,-12($sp)
        sw    $s1,-16($sp)
        move  $fp,$sp
        addiu $sp,$sp,-32
        ...

And so they also need to be restored at the bottom, like so:
        ...
        move  $a0,$s0
        jal   times100
        addu  $v0,$v0,$s1   # set the return value

        addiu $sp,$sp,32
        sw    $s1,-16($sp)
        sw    $s0,-12($sp)
        lw    $fp, -8($sp)
        lw    $ra, -4($sp)
        jr    $ra

### Caller-saved registers

We've been passed four arguments into `four_digits`, in the argument
registers `a0`-`a3`. But we need to make three function calls, and
two of those calls need to pass parameters using `a0` and `a1`.
So that means, if `four_digits` needed to use these two values
after it made the first function call, then it had better save them
somewhere.

This leads to another notion: "caller-saved registers." Before
any function is called, by convention the caller must save any
caller-saved registers before it makes the call. Those aren't
assumed to be preserved by the function being called, and so
it's the caller's job to save them.

But it turns out that after the first call to `two_digits`, 
the first two arguments to `four_digits` will no longer
be needed. The last two, held in `a2` and `a3`, will.
And then they need to be restored after the call.

        sw    $ra,-4($sp)
        sw    $fp,-8($sp)
        sw    $s0,-12($sp)
        sw    $s1,-16($sp)
        move  $fp,$sp
        addiu $sp,$sp,-32

        sw    $a2,-20($fp)
        sw    $a3,-24($fp)
        jal   two_digits
        move  $s0,$v0
        lw    $a0,-20($fp)
        lw    $a1,-24($fp)
        jal   two_digits
        move  $s1,$v0

        move  $a0,$s0
        jal   times100
        addu  $v0,$v0,$s1

        addiu $sp,$sp,32
        sw    $s1,-16($sp)
        sw    $s0,-12($sp)
        lw    $fp, -8($sp)
        lw    $ra, -4($sp)
        jr    $ra

The above is the code we need. Note that we did not restore
`a2` and `a3` back to those same registers. Since they are
the two arguments to the second call to `two_digits`, we just 
put them directly into `a0` and `a1` instead and made the
second call.

Note the use of `fp` instead of `sp` here. That's because we 
moved `sp` downward and put `fp` in its place.

Note that we don't have to worry about $s0$ and $s1$ getting
clobbered. They are callee-saved, so we can trust that,
should `two_digits` or `times100` use them, they will
save and restore them for us.

But note that we happen to know that `a2` and `a3` aren't
toucjed by the first call. Nevertheless, by convention,
we save and restore them.

### Caller-saved temporaries

By the MIPS calling conventions, `a0`-`a3` are caller-saved
registers. It turns out that the temporaries `t0` and `t1`
are caller-saved registers, too. If, instead of using `s0`
and `s1` we instead used `t0` and `t1`, then the code we'd
generate would instead be

        sw    $ra,-4($sp)
        sw    $fp,-8($sp)
        sw    $s0,-12($sp)
        sw    $s1,-16($sp)
        move  $fp,$sp
        addiu $sp,$sp,-32

        sw    $a2,-20($fp)
        sw    $a3,-24($fp)
        jal   two_digits
        move  $t0,$v0
        
        sw    $t0,-28($fp)
        lw    $a0,-20($fp)
        lw    $a1,-24($fp)
        jal   two_digits
        move  $s1,$v0

        lw    $a0,-28($fp)
        jal   times100
        addu  $v0,$v0,$s1

        addiu $sp,$sp,32
        sw    $s1,-16($sp)
        sw    $s0,-12($sp)
        lw    $fp, -8($sp)
        lw    $ra, -4($sp)
        jr    $ra

In the above, we add a 7th slot in the frame to temporarily store
`t0` before the call to `two_digits`. And then we restore it directly
as the parameter for the call to `times100`.

### Last conventions

Recall that I assumed that the frame needed was 32 bytes in size.
And note that we only used 24 bytes.  Some references on MIPS
say that a frame must be at least 32 bytes in size. So we have
made it 32.

I'd also like to point out that, if we keep byte or half-word values, 
then you could imagine building a frame of size that is not divisible
by 4.  It turns out, though, that MIPS32 frames must be "word-aligned"
this means that they must be broken into four byte chunks, and also
that the frame and stack pointer addresses must be divisible by 4.

I've read in some references, in anticipation of 64-bit words,
that these two pointers furthermore must be divisible by 8. They
need to be "double-word aligned."

Lastly, I've read in some references that the caller needs to save space
for the callee to save the passed arguments `a0`-`a3`.  So that means
that, following this convention, that the callee can use the bytes at
or above its frame pointer to save its argument registers. And that 
also means that a caller has to put space for on its stack for 
arguments before it makes a call to the callee. I've chosen not
to show this convention in the code above.

I also haven't mentioned what should be done when a function
takes more than four arguments. These extra arguments should be pushed 
onto the stack just before the call. Or, extending the convention just mentioned,
the bytes at or above the caller's stack pointer (i.e. just at or above
the callee's frame pointer) hold the 5th etc. arguments, but there needs to
be four slots reserved just above those.

---

## Summary

• Before a function is called the caller must

1. Save all the caller-saved registers it needs after the call.
These are `a0`-`a3` and `t0`-`t9`.

2. Put arguments (up to 4) into registers `a0`-`a3`. Any additional
arguments should be placed at the boottom of its stack frame.

3. Use `JAL` with the label of the first instruction of the callee.

• At the start of a call the callee must

1. Save the callee-saved registers `ra`, `fp, `sp`, `s0`-`s7`
into its stack frame.

2. Move `sp` to the bottom of its stack frame, and have `fp` where `sp` was.

• Before the callee returns it must

1. Set `v0` to the return value.

2. Restore all the registers it saved at the start, including the return address `ra`

3. Use `JR` with `ra`

• After the caller should

1. Restore any registers it saved.

2. Undo any extra arguments it pushed onto the stack before the call.

3. Extract the value returned in `v0`.

*Whew.*
