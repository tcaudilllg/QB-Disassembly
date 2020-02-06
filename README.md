# QB-Disassembly
Notes and patches for Microsoft QBASIC for DOS


This git will host notes and observations QBASIC's source code, which was leaked a few years back along with MS-DOS 6. The source is property of Microsoft and will not be hosted here, but is available from several sources... search for it.


# OVERVIEW

QBASIC (code name "QB5") consists of several projects, each with their own purpose:
- BASCOM is the original BASIC compiler (original version for IBM PC is 5, however some of the code goes back farther).
- COW (Character Oriented Windows) provides API and shell for the user interface. It is very similar to Microsoft Windows, but for text mode.
- BQLB is the runtime library.

QBASIC is a synthesis of BASCOM and COW, with BQLB as the backend.


# Structure of BASCOM

BASCOM, which runs alongside COW, consists of several stages. The first stage is the argument scanner, which takes directives from the user via commandline (if any) for special operation. The second stage is contextualizing, which happens at program load and is the process of making QB aware of modular code, for features like compartmentalizing SUBs, FUNCTIONs, and modules for independent viewing. The third stage is tokenizing, which breaks the code lines into their constituent parts. The fourth stage is parsing, which attempts to determine the relationships between tokens. The parser emits an intermediary form of code called "bytecode" which allows QB to interpret the program faster. After parsing comes the fifth stage, tabling, which identifies relationships between lines of code and produces additional bytecodes marking these. The 6th stage is scanning, which consists of minor (but important) tweaks that taken together make coding style more flexible for users. The scanner also looks for errors in code. The scanner sends its proofred codes to the executor which, depending on the mode, enacts it or assembles machine code in its likeness, possibly but not necessarily through the runtime.

Code hosted in the interpreter is dealt with one line at a time. It is tokenized, parsed, then "listed" according to the rules for what the compiler assumes it represents. Listing capitalizes reserved words and enforces white spacing. No listing will occur if syntax checking is disabled. Listing is forced upon the user when loading code saved as binary, as it is the method by which the code is converted back to text and made readable.

# Editing BASCOM

BASCOM relies on lookup tables which are auto-generated from markup files by dedicated tools.
