Project to improve the speed of POKE.

2/8/2020 12:00AM EST
First attempt:
Tried to strike the scan rule coercion from the runtime dispatch.
`exstmisc.asm`, line 89
```
MakeExe	exStPoke,opStPoke
	;pop	bx				;preserve value to be poked
	;call	I4toU2				;coerce I4 addr on stack to a U2
	;push	bx				;(I4toU2 preserves bx)
	CALLRT	B$POKE,Disp
```
Test program:
```
' Half screen fill in mode 13h
' Expected results: fill half the screen with blue
Screen 13: DEF SEG = &HA000:
For index% = 0 to 31999

    POKE index%, 1
    
Next
```
Results:
About four lines filled with blue, then program end. Environment became unresponsive.

Here's the code from the coercer (exio.asm, line 40):
```
;***
;I4toU2
;
;Purpose:
;
;   Given an I4 on the stack (before near return address), coerce it to
;   an equivalent U2, replacing the I4 with the U2.
;   Runtime error if overflow.
;
;Preserves:
;
;   bx
;
;****

	public	I4toU2
I4toU2	proc	near
	pop	cx			;near return address
	pop	ax
	pop	dx			;dx:ax is the I4
	push	ax			;assume success
	ror	dx,1			;put low bit of high word in PSW.C
	adc	dx,0			;set PSW.Z if PSW.C == sense of all bits
	jnz	@F			;brif overflow error
	jmp	cx			;return

@@:
	jmp	exMathErrOVF		;Declare overflow error
I4toU2	endp

```

All this just to write a byte. Starting to look like a 10x penalty, yet? The code suggests that the POKE opcode takes a 4-byte integer (`I4`) for the offset, which is twice the bytes needed (an unsigned 2-byte integer (`U2`) would do). It would be great if we could eliminate QB's useless overflow exception handling and dispense with the use of this routine altogether for POKE. Half this routine is error handling, so let's see if we can eliminate that. Perhaps if we integrated the routine into the POKE executor itself? It seems like we may have screwed with the stack by striking I4toU2, so let's try to handle it at exStPOKE. If we suceed, we'll increase POKE's efficiency by around 30%.

Or so I thought... the strategy works, but the speedup is a more modest 15%. There is much more going on behind the scenes than simple stack management. I tried to move the `POKE` code itself out the runtime for a big speedup, but just ended up with crashes. I ended up running QBASIC thru DOSBOX debugger to get a birds eye view of what was happening. A review of the logs shows that the executor runs every line through a ringer of overflow detections and other debugger suite goodies (none of which is even made available to QBASIC). I managed to route around this by replacing the dispatch runtime routine with a simple fetch/jump to the next intruction (ever waiting at the pointer in `ds:si`), resulting in a massive performance increase of 200%. I managed further improvements by applying similar changes to the rest of the routines that didn't rely on the runtime, such as 16-bit math and variable value copies from bytecode to the stack. I came to realize that 32-bit arithmetic should not to be used in performance critical code on 16-bit machines... The 8086/286 processors have no native support for it, forcing QB to call its runtime to execute 80 lines of code or more to do the job, at a cost of 200 or more cycles depending on the operation.

While looking over the logs I also found this interesting performance boondoggle.
```
23FC:00002B05  es:lodsw                                                26 AD                 EAX:00002B05 EBX:0000FF40 ECX:00006E1E EDX:00006E1E ESI:0000004C EDI:00007EAA EBP:00007E6E ESP:00007E64 DS:3EC4 ES:6E1E FS:0000 GS:0000 SS:3EC4 CF:0 ZF:1 SF:0 OF:0 AF:0 PF:1 IF:1 TF:0 VM:0 FLG:00007212 CR0:00000000
23FC:00002B07  inc  ax                                                 40                    EAX:00000015 EBX:0000FF40 ECX:00006E1E EDX:00006E1E ESI:0000004E EDI:00007EAA EBP:00007E6E ESP:00007E64 DS:3EC4 ES:6E1E FS:0000 GS:0000 SS:3EC4 CF:0 ZF:1 SF:0 OF:0 AF:0 PF:1 IF:1 TF:0 VM:0 FLG:00007212 CR0:00000000
23FC:00002B08  and  ax,FFFE                                            25 FE FF              EAX:00000016 EBX:0000FF40 ECX:00006E1E EDX:00006E1E ESI:0000004E EDI:00007EAA EBP:00007E6E ESP:00007E64 DS:3EC4 ES:6E1E FS:0000 GS:0000 SS:3EC4 CF:0 ZF:0 SF:0 OF:0 AF:0 PF:0 IF:1 TF:0 VM:0 FLG:00007212 CR0:00000000
23FC:00002B0B  add  si,ax                                              03 F0                 EAX:00000016 EBX:0000FF40 ECX:00006E1E EDX:00006E1E ESI:0000004E EDI:00007EAA EBP:00007E6E ESP:00007E64 DS:3EC4 ES:6E1E FS:0000 GS:0000 SS:3EC4 CF:0 ZF:0 SF:0 OF:0 AF:0 PF:0 IF:1 TF:0 VM:0 FLG:00007212 CR0:00000000
23FC:00002B0D  es:lodsw                                                26 AD                 EAX:00000016 EBX:0000FF40 ECX:00006E1E EDX:00006E1E ESI:00000064 EDI:00007EAA EBP:00007E6E ESP:00007E64 DS:3EC4 ES:6E1E FS:0000 GS:0000 SS:3EC4 CF:0 ZF:0 SF:0 OF:0 AF:1 PF:0 IF:1 TF:0 VM:0 FLG:00007212 CR0:00000000
23FC:00002B0F  jmp  near ax                                            FF E0                 EAX:00002A42 EBX:0000FF40 ECX:00006E1E EDX:00006E1E ESI:00000066 EDI:00007EAA EBP:00007E6E ESP:00007E64 DS:3EC4 ES:6E1E FS:0000 GS:0000 SS:3EC4 CF:0 ZF:0 SF:0 OF:0 AF:1 PF:0 IF:1 TF:0 VM:0 FLG:00007212 CR0:00000000
```

The count of these bytecodes and their positions suggested to me that they were codes for blank lines. Inspecting the executor confirmed the fact. Apparently the interpreter must cohabitate with the lister, using the same virtual program code, to save memory, resulting in the necessity of treating both line breaks and comments as statements in their own right, with all the attendant overhead. This isn't the only place where the interpreter must take pains to step over information required by the lister, but further edits throw off the lister and cause the IDE to fail.

When my changes were complete (to QB and to my own code) I had acheived performance improvements of 450% (see `Speed Test: Half-Screen Fill` for more info).
