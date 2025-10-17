# yw-cond
A web-based W.I.P Cond analyser for the Yo-kai Watch series.
This parser is for the Yo-kai Watch franchise which has a more complex cond set (and some slight changes) from Inazuma Eleven, for IEGO cond parsing take a look at [level5_condition](https://github.com/Tiniifan/level5_condition/) made by Tiniifan themself!
[The website can be downloaded or viewed here](https://n123git.github.io/yw-cond).

## Cond Timeline
* IEGO (2011, first cond system known; extremely simplistic)
  ->
* YW1 (2013, lorem ipsum)
  ->
* YW2 (2014) Complex, multi-layered container format with support for functions acting as multi-dimensional arrays.
->
* (Hasn't really changed much since but are prevelant in later games like YW4 and Snack World)

## Cond Documentation
Conds are a base64-encoded serialised, binary format used to contain and represent conditions within level5 games. The below documentation documents the known aspects of said format, ~~and will not be updated regularly~~:

Conds are encoded in base64 - this is most likely as strings are the only variable length data type in `cfg.bin`s (via the use of string tables) - where they are stored.

After decoding to binary, which can be done with a website like [this](https://cryptii.com/pipes/base64-to-hex); the resulting cond will *always* start with a 4 byte header of `00 00 00 00`.

After that, you will have a 2 byte reigon known as the `COND_CODE`; the first byte of this will represent the amount of bytes (not including itself but it includes the other byte of `COND_CODE`) until the end of the cond in question - in certain cases where several conditions are stored in one cond this will be `F0` (240) instead.

Then you have 2 operations
* `READ_LITERAL` - this defines a 4-byte, little-endian literal value called `COMPARISON_VALUE`.
* `READ_MEMORY` - this pulls a value from memory using 4-byte `RESOURCE_ID`s, with the type being defined by a 3 byte `CTYPE`.
Common known `CTYPE`s include (but are not limited to!):
* `00 06 02` - An Integer, as the system dosen't distinguish between bools and ints this is also used for boolean values.
* `00 01 00` - This is used as a Special Value, but is most likely for zero parameter functions i.e. `GetWatchRank()`.
* `00 0A 01` - A function with one parameters - this is used for several reasons, a common example is shown below (this example checks if the player has the Item `0x12345678`:
```md
# HEADER (header + condcode)
HEADER: 00 00 00 00
COND_CODE: 18 05
# CONDITION
READ_MEMORY: 35
RESOURCE_ID_A: 8D 76 66 D8 — Check if player has Item (YW2)
CTYPE_A: 00 0A 01 — Function
EXTENSION_DELIM: 28
CTYPE_B: 00 06 02 — Int
READ_SUBSECTION: 34
RESOURCE_ID_B: 78 56 34 12
READ_LITERAL: 32
COMPARISON_VALUE: 00 00 00 01 — true
OPERATOR: 78 — ==
```
or in psuedo-C:
```c
// #header 00 00 00 00
// #condCode 18 05

if (HasItem(0x12345678) == 0x1) {
    success();
}
fail();
```
Another key use of `00 0A 01` is Arrays/Lists; since the system dosen't natively hold the ability to handle arrays or any data type of variable-length, the game will frequently define functions like `GetXYZ(int: index)` which emulates array access.

An example of this can be found below:
```md
# CONDITION
READ_MEMORY: 35
RESOURCE_ID_A: 77 B4 63 E5 — Grab Psychic Blasters Boss IsCleared (YW2)
CTYPE_A: 00 0A 01 — Function
EXTENSION_DELIM: 28
CTYPE_B: 00 06 02 — Int
READ_LITERAL: 32
COMPARISON_VALUE: 00 00 00 02
READ_LITERAL: 32
COMPARISON_VALUE: 00 00 00 01 — true
OPERATOR: 78 — ==
```
This in psuedo-c is effectively equivalent to:
```c
if (IsCleared[0x02] == 0x1) {
  success();
}
fail();
```
