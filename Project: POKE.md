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

Here's the code from the coercer:
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
