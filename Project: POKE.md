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
