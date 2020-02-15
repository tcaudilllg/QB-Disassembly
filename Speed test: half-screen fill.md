# Speed tests: QB versions compared

Here I compared four versions of QB: QBASIC, QB4.5, QB7 PDS, and QBFaster handling half-screen fill while running in IDE interpreted mode. Test system was DOSBOX (386) at 1000 cycles, 8 megs RAM, core=dynamic.

Here's the code:
```
SCREEN 13
time1 = TIMER
DEF SEG = &HA000
FOR index% = 0 TO &H7CFF
POKE index%, 15
NEXT
PRINT TIMER - time1
SLEEP
```
#### Results (in seconds, lower is better)

* QBASIC: 8
* QB 4.5: 3.1
* QB 7.1: 2.36
* QBFaster: 1.7

And here's the same result using `Line (0,0)-(319,99),15,bf` instead of the loop: < .05 (it appeared instantaneous). Never underestimate the power of REP MOV ;)

Compiled code with BC6 (optimized for 8086) was alleged by the timer formula to complete the fill in 0.28 secs, but it happened too slow for that to be accurate. That I was able to apprehend the draw as it happened suggests it took just under a second. This result is inline with what seems to be the fundamental penalty for interpreted bytecode for atomic operations on the x86 architecure (50%).

Unrolling the loop resulted in faster code (25% at 16 unrolls). Moving the `Def Seg` call before the timer init resulted in a .05 sec improvement.

#### Why so slow?

Several reasons:
* the atomic, space-saving nature of the interpreter leaves little room for complex analytic algorithms. Indeed, the results suggest that the compiler does no compile-time optimization at all. Instead, the compiler just copies code from the runtime according to what's interpreted while passing mondo parameters on the stack. Technically there is no compiler to speak of... just a substitution algorithm of ASM for bytecode. That the interpreter basically is the compiler causes it to get in the way of itself in the performance of its role, forcing it to rely on the stack to transfer data between seperate statements and expressions. It's the equivalent of using ASM or C with subcalls on every line for every little thing, the result being what could have been done in a few instructions ends up being done in dozens.
* QB isn't designed for performance applications; rather, it was created to produce near-perfect, bug-minimal, fault-tolerant code, all of which it does quite well.
* It was made by close friends of Bill Gates who shared his affections for fun, quirky interfaces which de-emphasized performance (these are people who witnessed with their own eyes the birth of computer graphics; for such people the mere sight of animation on a computer monitor is a source of endless fascination and is entertainment on its own merit).
