# QBASIC Disassembly
Notes and patches for Microsoft QBASIC for DOS


This git will host notes and observations regarding QBASIC's source code, which was leaked a few years back along with MS-DOS 6. The source is property of Microsoft and will not be hosted here, but is available from several sources... search for it.


## OVERVIEW

QBASIC (code name "QB5") consists of several projects, each with their own purpose:
- `BASCOM` is the original BASIC compiler (original version for IBM PC is 5, however some of the code goes back farther). Its code is considered "version independent", and is in folder "`QB`".
- `COW` (Character Oriented Windows) provides API and shell for the user interface. It is very similar to Microsoft Windows, but for text mode. Its code is in folder "`BEEF`", and is built in folder "`COW`".
- `BQLB` is the runtime library. It resides in folder "`RUNTIME`".

QBASIC is a synthesis of `BASCOM` and `COW`, with `BQLB` as the backend. Its glue code is in folder "`45\QB5\QB`" and is built to "`45\QB5\QBRUN`". To build it using the tools in "`45\TL\BIN`", navigate to "`45\QB5\QBRUN`" in DOS and enter "`SAMPLE`". (hint: it helps to edit `SAMPLE.BAT` first, replacing "`$(QB)`" with "`45`". Also, be sure to move the root folder, "`45`", to root of drive). Building in latest release of QEMU hosting FreeDOS crashes QEMU; recommend DOSBOX. Builds in 2min at 200000 DOSBOX cycles or equivalent.


## Structure of BASCOM

`BASCOM`, which runs alongside `COW`, consists of several stages. The first stage is the argument scanner, which takes directives from the user via commandline (if any) for special operation. The second stage is contextualizing, which happens at program load and is the process of making QB aware of modular code, for features like compartmentalizing `SUBs`, `FUNCTIONs`, and modules for independent viewing. The third stage is tokenizing, which breaks the code lines into their constituent parts. The fourth stage is parsing, which attempts to determine the relationships between tokens. The parser emits an intermediary form of code called "bytecode" which allows QB to interpret the program faster. After parsing comes the fifth stage, tabling, which identifies relationships between lines of code and produces additional bytecodes marking these. The 6th stage is scanning, which consists of minor (but important) tweaks that taken together make coding style more flexible for users. The scanner also looks for errors in code. The scanner sends its proofred codes to the executor which, depending on the mode (interpreter or compiler), enacts it or assembles machine code in its likeness, possibly but not necessarily through the runtime.

Code hosted in the interpreter is dealt with one line at a time. It is tokenized, parsed, then "listed" according to the rules for what the compiler assumes it represents. Listing capitalizes reserved words and enforces white spacing. No listing will occur if syntax checking is disabled. Listing is forced upon the user when loading code saved as binary, as it is the method by which the code is converted back to text and made readable.

## Editing BASCOM

BASCOM relies on lookup tables which are auto-generated from markup files by dedicated tools.

### Grammar table (QBASBNF.PRS)

`QBASBNF.PRS` contains four tables. The first is the list of all reserved words known to the interpreter. They are of the form:

`tk[label] ("[word]")`

Reserved words are automatically capitalized by the environment's lister component.

The second and third tables define the grammar. For a reserved word to be useful, it must have at least one grammar. The parser uses the grammar to determine which bytecodes to create. Reserved words may have multiple grammar, each meaning a different code. The second table consists of statement grammer, followed by function grammar. Statements and functions may produce multiple bytecodes. Grammar entries have the form

`tk[label] [Exp | tkComma | MARK([mark id]) | tk[reserved word]] EMIT([opcode label]);`

Grammar roughly conforms to the syntax described in the online help files, and uses the same markup. Of particular interest is the `MARK` symbol, which can help the parser/scanner make sense of complex grammar (with a little help from the programmer).

`EMIT` is the command to create a specific opcode for the grammar. There must be an entry in the table in `PEROPCOD.TXT` for each opcode label used in `QBASBNF.PRS`.

Following the grammar is the list of symbols used by the grammar, and combinations of these with their own labels.

Running `QBASBNF.PRS` through `BNFPRS.EXE` produces "binding" ASM include files which directly integrate the grammar spec into the compiler/interpreter.


### Rule table (PEROPCOD.TXT)

Each opcode is bound by a set of rules specific to it. These rules are specified in PEROPCOD.TXT. The following is an example ruleset
from the table therein:
```
opStPset	  |     		  | Ss_0_0	      | 0
                    exStPset              | 0+OPA_fExecute
                    LrStPset              | ORW_Pset;
```

This ruleset observes the following form:
```
[opcode label] |  [feature flag]  	  | [scanrule label]	      | [coercion formula] 
                    [executor label]              | [attribute count]
                    [list rule label]              | [reserved word id]
```

Fields are terminated by either `\` or line break.

* `Opcode label` begins with `Op`. It is the identifier for the opcode as passed to EMIT in the grammar (as described in `QBASBNF.PRS`).
* `Feature flag` -- if this field is filled and its text prefixed by an asterisk, then the opcode and its grammar will not be present in the final build (else it has no effect). This is a line of demarcation between cripple builds like QBASIC vs feature complete builds like QuickBASIC 4.5.
* `Scanrule lable` begins with `Ss`. It is a directive to the scanner to apply particular proofreeding rules using dedicated routines. The above example uses `rule 0_0`, which is essentially a "do nothing" rule. The scanner and its rules are described in the files prefixed with `ss`.
* `Coercion formula` is a further directive to the scanner in the context of the scanrule. It specifies rules for the transformation of data between types, if necessary, in preparation for its dispatch to the runtime (example would be converting the offset parameter passed to `POKE` from a signed 32-bit long integer offset to an unsigned 16-bit short integer ("I4 -> I2")). The above example uses `rule 0`, which implies default behavior of no coercion.
* `Executor label` begins with `ex` and is a reference to the executable macro which enacts the bytecode's purpose (in this case, drawing a pixel using QB's crash-proof `Pset` runtime subroutine). The executor macro set is spread across the files prefixed with `ex`.
* `Attribute count` helps the interpreter know how many descriptive bytes to package along with the opcode (for making opcode-dependent deductions described in ASM). This information is used to some extent by the tabler, which is spread around files prefixed with `txt`.
* `List rule label` begins with `Lr`. It describes to the lister how to re-emit text after the scanner gobbles it. This rule must be compatible with the grammar, or else source code will be mangled automatically by the environment.
* `Reserved word id` is a reference to the reserved word the opcode , as described in `QBASBNF.PRS`. It is only used internally and is managed completely by the build process.

The existence of `Attribute count`, and its nebulous definition in the source notes, is the first clue that these files do not constitute a "BASIC interpreter kit" by which new commands can be made without knowledge of ASM. Mixing and matching rules between opcodes results strange compile-time errors in all but the barest of cases. A single word statement with no parameters can be created by specifying list rule `LrRw`, scan rule `SS_0_0`, coercion formula `0`, and attribute count `0+OPA_fExecute`. Anything more complicated is likely to prompt QB to make strange complaints about "WHILE without WEND" or "SELECT without END".
