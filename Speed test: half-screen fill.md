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
