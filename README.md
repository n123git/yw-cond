# yw-cond
A web-based work-in-progress UI and toolkit for parsing, decompiling, analyzing and generating Yo-kai Watch Conds (CExpressions), with frequent updates.

This parser is for the Yo-kai Watch franchise which has a much more complex system (and some slight changes) from Inazuma Eleven: GO, for IEGO Cond parsing take a look at the newly released [level5_condition](https://github.com/Tiniifan/level5_condition/) made by Tinifan themself!

Examples of how the games use Conds can be shown below:
* Yo-kai AI
* NPC/Treasure Chest Availability
* Quest/Trophy Conditions
* Shop Item/Price Conditions
* Story Progression
* NPC Interaction handler
* Soundtracks
* Minimap Image
* Mirapo Unlocks
* Face Icon Unlocks
* Treasure Chest Loot Tables
* Fishing/Searching Loot Tables
* Weather
* Trains
* Springdale Scratch Off
* and much, much more!

[The website can be downloaded or viewed here](https://n123git.github.io/yw-cond).

v1.4b release progress and changelog:
* Float support - 100%
* Jump support - 30%
* Rewritten cond parser to fix remaining bugs - 90%
  * Sub-block parsing isn't fully correct yet, runs under old assumptions caused by level5's severe underuse of their own system's capabilities.
* Readability and maintainability improvements - 91%
* A cond compiler to create entirely new conds with support for floats, brackets, implicit ordering, scientific notation, advanced recursive subsections and more - 90%
  * Jumps aren't supported yet - might implement shorthands like range(x, y) for >= x && =< y
* ~~Improved light mode - 0%~~
  * Does any sane person legitametly use this? /j
  * Delayed to v1.41b as the scope of the update is getting too large and this will take a lot of time and CSS changes for no benefit to 100% of people using this tool
* More config options for safety - 100%
* More labelled CExpression funcs - 100%
* Clean up UI - 90%
* Add more templates - 100%
* More decompiler improvements (aggressive simplification) - 0%

# CExpression (Cond) System Documentation

The **CExpression system**, is a proprietary (usually Base64-encoded) system used for evaluating recursive RPN (Reverse Polish Notation) runtime conditions known as Conds. These Conds are evaluated by `CExpression::CalcSub` internally and contain *one* or more conditions which consist of:
  * Literal values (Constants, IDs etc)
  * Functions (Engine calls)
  * Operators (arithmetic, logical and bitwise)
  * Jumps (conditional and unconditional)
    * These jumps can only move forward and therefore can only be used for if else, NOT complex control flow such as loops. This sadly means the CExpression system is *NOT* turing-complete.
> Note: conditions have been found which contain < 1 conditions but these are invalid and are the reason for the music app being broken in localised versions of Yo-kai Watch 2: Psychic Specters.

> Note: within the following documentation assume all numbers encased in a code block i.e. `02` are in hexadecimal (base-16) unless otherwise specified.
## 1. **Cond Structure**
Each Cond begins with a header section composed of a 3-byte header of `00 00 00`, and a section previously referred to as the `COND_CODE`. This is composed of a uint16 `COND_LENGTH` and a uint8 `STACK_PRM`.
> This header will always be `00 00 00` in the Cond itself; the engine will fill it in with the appropriate data during parsing.

### 1.1 Header (3 bytes)
The header is a 3 byte region at the start of a Cond - this should *always* be `00 00 00` in the actual Cond itself - the engine will process it accordingly.

### 1.2 Cond Code (3 bytes)
a `COND_CODE` is the old name given to a 3-byte (previously thought to be 2-byte) section of a Cond's header now known as two separate parts the uint16 `COND_LENGTH` and uint8 `STACK_PRM.` Sections describing the layout of this section can be found below:

#### 1.2.1 COND_LENGTH
This uint16 defines the total length excluding itself (and all prior bytes) but including all subsequent bytes within the Cond - including the `STACK_PRM`.

#### 1.2.2 STACK_PRM
This byte known as `STACK_PRM` represents the Top-Level expression count - meaning the amount of elements in a Cond excluding subsections and their markers (CTYPEs) - where elements in a subsection contribute to the CTYPE (See § 6) of the subsection. `STACK_PRM` is incremented by every element after itself and outside of a subsection - with the exception of CTYPEs which don't contribute to `STACK_PRM` as they are, in of themselves a subsection marker which host their own `STACK_PRM` and `COND_LENGTH` (See § 6). This includes everything from operators to literal values to `READ_LITERAL`s and more!
```hex
00 00 00
00 36
05
35 <- READ_FUNCTION; contributes 1 to STACK_PRM
74 03 A9 CE <- LITERAL_VALUE; contributes 1 to STACK_PRM
00 1C 03 <- CTYPE; subsections do not contribute to the STACK_PRM of the main cond
 28 
  00 06 02 <- CTYPE in a subsection subsection; does not contribute
  34 <- READ_HASH contributes 1 to the subsection subsection's CTYPE
  C1 B2 DA B7 <- LITERAL_VALUE contributes 1 to the subsections subsection's CTYPE
 28 <- READ_PARAM in a subsection; contributes 1 to the subsection's CTYPE
  00 06 02 <- CTYPE in a subsection subsection; does not contribute
  34 <- READ_HASH contributes 1 to the subsection subsection's CTYPE
  8E 31 15 F3 <- LITERAL_VALUE contributes 1 to the subsections subsection's CTYPE
 28 <- READ_PARAM in a subsection; contributes 1 to the subsection's CTYPE
  00 06 02 <- CTYPE in a subsection subsection; does not contribute
  32 <- READ_LITERAL contributes 1 to the subsection subsection's CTYPE
  00 00 0E F6 <- LITERAL_VALUE contributes 1 to the subsections subsection's CTYPE
35 <- READ_FUNCTION; contributes 1 to STACK_PRM
69 84 E3 AF <- LITERAL_VALUE; contributes 1 to STACK_PRM
00 0A 01 <- CTYPE; subsections do not contribute to the STACK_PRM of the main cond
 28 <- READ_PARAM in a subsection; contributes 1 to the subsection's CTYPE
 00 06 02 <- CTYPE in a subsection subsection; does not contribute
 34 <- READ_HASH contributes 1 to the subsection subsection's CTYPE
 42 6F A0 C3 8F <- LITERAL_VALUE contributes 1 to the subsections subsection's CTYPE
```

> Example: `00 00 00 - 00 0F - 05 - 35 10 B1 40 96 00 01 00 32 00 00 00 01 78`

---

## 2. **Data Identifiers**

### 2.1 Reads

| Type            | Hex Code | Decimal Code | Description                                                                                     |
| --------------- | -------- | ------------ | ----------------------------------------------------------------------------------------------- |
| READ_PARAM      | `28`     | `40`         | Reads a param within a `READ_FUNCTION` call. Followed by a CTYPE.                               |
| READ_LITERAL    | `32`     | `50`         | Pushes an integer of type 4.                                                                    |
| READ_FLOAT      | `33`     | `51`         | Pushes an IEEE 754 float of type 6.                                                             |
| READ_HASH       | `34`     | `52`         | Similar to a READ_LITERAL but is used to push a hash instead. Internally functions identically. |
| READ_FUNCTION   | `35`     | `53`         | Reads and pushes a function. Followed by a CTYPE representing the function as a whole.          |

> Note that the reads aside from `READ_PARAM` as it works differently, follow a limit of a maximum of 64 stack values at a time.

## 3. **Operators**

Operators pop an arbitrary number of values off the stack, equivalent to it's param count, perform a logical, arithmetic or bitwise operation on those values and then push the output to the stack. Operators pop the *most recent* values off of the stack (LIFO: Last In, First Out) as is common for similar systems.
| Hex Code | Decimal | Symbol | Op Count | Operation                             | Notes                                                                                                  |
| -------- | ------- | ------ | -------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `46`     | `70`    | ++     | 1        | Incrementation                        | Unofficial Symbol.                                                                                     |
| `47`     | `71`    | --     | 1        | Decrementation                        | Unofficial Symbol.                                                                                     |
| `50`     | `80`    | ~      | 1        | Bitwise NOT                           | Unofficial Symbol.                                                                                     |
| `51`     | `81`    | !!     | 1        | To Bool                               | Returns 1 if != 0 else 0. Unofficial Symbol.                                                           |
| `5A`     | `90`    | *      | 2        | Multiply                              |                                                                                                        |
| `5B`     | `91`    | /      | 2        | Divide                                |                                                                                                        |
| `5C`     | `92`    | %      | 2        | Modulus                               |                                                                                                        |
| `5D`     | `93`    | +      | 2        | Addition                              |                                                                                                        |
| `5E`     | `94`    | -      | 2        | Subtraction                           |                                                                                                        |
| `64`     | `100`   | <<     | 2        | Left shift                            |                                                                                                        |
| `65`     | `101`   | >>     | 2        | Right shift                           |                                                                                                        |
| `6E`     | `110`   | <      | 2        | Less than                             |                                                                                                        |
| `6F`     | `111`   | <=     | 2        | Less or equal                         |                                                                                                        |
| `70`     | `112`   | >      | 2        | Greater than                          |                                                                                                        |
| `71`     | `113`   | >=     | 2        | Greater or equal                      |                                                                                                        |
| `78`     | `120`   | ==     | 2        | Equal                                 |                                                                                                        |
| `79`     | `121`   | !=     | 2        | Not equal                             |                                                                                                        |
| `82`     | `130`   | &      | 2        | Bitwise AND                           |                                                                                                        |
| `83`     | `131`   | \|     | 2        | Bitwise OR                            |                                                                                                        |
| `84`     | `132`   | ^      | 2        | Bitwise XOR                           |                                                                                                        |
| `8F`     | `143`   | &&     | 2        | Logical AND                           | By far the most common.                                                                                |
| `90`     | `144`   | \|\|   | 2        | Logical OR                            | Frequently combined with `8F`.                                                                         |

Operators are grouped by the multiple of ten in their decimal opcode. Each `x0–x9` range represents a distinct operator class, and operators within a range are closely related in behaviour as shown by the table below:

| Range | Category                | Operators Included      |
| ----- | ----------------------- | ----------------------- |
| `7X`  | Unary Arithmetic        | `++`, `--`              |
| `8X`  | Unary Bitwise / Logical | `~`, `!!`               |
| `9X`  | Binary Arithmetic       | `*`, `/`, `%`, `+`, `-` |
| `1X`  | Bit Shifting            | `<<`, `>>`              |
| `11X` | Relational Comparison   | `<`, `<=`, `>`, `>=`    |
| `12X` | Equality Comparison     | `==`, `!=`              |
| `13X` | Bitwise Binary Logic    | `&`, `\|`, `^`          |
| `14X` | Logical Binary Logic    | `&&`, `\|\|`            |

## 3.1 Jumps
There are two kinds of jumps supported by the CExpression engine, these can be considered as pseudo-ops as they allow conditional or optional execution of sub-blocks. Note that these jumps are *forward-only*, meaning they can be used to express `if` / `if-else`–style logic, but sadly *cannot* implement loops or any other arbitrary control flow.

| Hex Code | Decimal Code | Symbol  | Operation                             | Notes                                                                                                  |
| -------- | ------------ | ------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `96`     | `150`        | ?->     | Conditional Jump Operator             | Unofficial symbol.                                                                                     |
| `97`     | `151`        | ->      | Unconditional Jump Operator           | Unofficial symbol.                                                                                     |

Both jump opcodes are immediately followed by a *CType* (See § 6 - CTypes), which determines both the length of the sub-block to skip or execute (aka how many bytes from the end of the CType to jump past) using the `DataSize` and a signed flag byte (`ExtData`) that further controls execution behaviour.

Note that improperly aligned jump lengths may cause the evaluator to skip valid instructions or misinterpret data as opcodes. Unknown opcodes are safely skipped, but malformed jumps can still lead to unintended behaviour so yeah :/
Additionally, as of writing this `yw-cond` doesn't support these in decompilation, recompilation or parsing so you'll have to manually test and generate them.

### 3.1.1 Conditional Jump
`0x96`/`150` (`?->`) acts as a conditional execution block, similar to an if/if ... else statement.

It pops one operand from the stack just like any other unary operator (we will call this the stack value for now due to how many values ?-> relies on) and reads its own *CType*. Where `DataSize` specifies the length (in bytes) of the sub-block and `ExtData` is interpreted as a signed flag. Which leads to the following branch of possibilities:
* If the stack value is truthy (!= 0) *and* the flag byte is > 0 (`0x01–0x7F`), the sub-block is executed via `CalcSub`, then jumped past.
* If the stack value is falsy (== 0), or the flag byte is < 1 (`0x00`/`0x80–0xFF`), the sub-block is jumped past without any execution or otherwise processing.

You can simplify this to the following psuedo:
```cpp
value = pop(); // pop a value off the stack, and grab it
ctype = readCType(); // simplified CType read because I'm lazy :p

if (value != 0 && ctype.ExtData > 0) { // != 0 is how level5 determines truthiness within all their systems
    CalcSub(ctype.DataSize);
}
skip(ctype.DataSize);
```

### 3.1.2 Unconditional Jump
`0x97` defines an *optional sub-block* whose execution depends solely on a flag and not on any runtime values being consumed or read, therefore we will consider it a nullary operator (0 operands).
When found the engine will read it's *CType* found right after the jump where `DataSize` gives the length (in bytes) of the sub-block and `ExtData` is treated as a signed flag.
This then leaves the engine with the following branch of possibilities:
* If the flag byte is > 0 (`0x01–0x7F`), the sub-block is executed via `CalcSub`.
* If the flag byte is < 1 (`0x00`/`0x80–0xFF`), the sub-block is jumped past entirely.

Despite being confirmed to have existed as early as Yo-kai Watch 1, these jumps are rarely used in practice. Although these can most likely be used to get around the 64 stack value limit in select situations.
Additionally since the game executes the Cond without an intermediary parsing stage you can place invalid bytes without consequence as long as it's jumped past and not executed i.e. using a `0 ->`  (falsy value + conditional jump).

## 4. **Read Operations**

### 4.1 READ_FUNCTION

A READ_FUNCTION entry is structured as shown below where they can have anywhere from 0-255 params:

```
READ_FUNCTION
  LITERAL_VALUE (4 bytes; is the CRC32 of the name of the function to execute)
  CTYPE (3 bytes; shows size and param count of the function)
  optional:READ_PARAM (1 byte)
     CTYPE (3 bytes; size and data of the below value)
     READ_HASH/READ_LITERAL/READ_FLOAT (1 byte)
     LITERAL_VALUE (4 byte)
  ... can recursively hold up to 255 READ_PARAMs
```

> Note: Like all values within Conds, Literal Values are *big-endian*.
<!-- Optional `READ_PARAM` chains  <!-- for a bit this was chins 😭 --\> allow for unlimited nesting of subsections for multi-param functions. -->

### 4.2 READ_LITERAL
* Always followed by a *signed 32-bit integer* (but can also be used for functions that expect a 16-bit, 8-bit, or Boolean value, in which case it should still padded to 4 bytes).
* Values are **big-endian**.
* Pushes an integer of type 4 (int32)

### 4.3 READ_FLOAT
* Pushes an IEEE-754 (standard float32) float of type 6 (float32)
* Very rare.

### 4.4 READ_HASH
* Pushes an integer of type 4 (int32)
  * Used for IDs hence the name `READ_HASH`

> Note that there is no runtime distinction between `READ_LITERAL` and `READ_HASH`.

## 5. CExpression Data Types
These data types are *never directly referenced in the Cond itself* but are used internally by the CExpression engine during evaluation and will therefore be mentioned here.
| ID | Type   | Description             |
| -- | ------ | ----------------------- |
| 0  | int8   | 8-bit signed integer    |
| 1  | uint8  | 8-bit unsigned integer  |
| 2  | int16  | 16-bit signed integer   |
| 3  | uint16 | 16-bit unsigned integer |
| 4  | int32  | 32-bit signed integer   |
| 5  | uint32 | 32-bit unsigned integer |
| 6  | float  | 32-bit floating point   |

## 6. CTypes
CTypes are **3-byte descriptors** which act as the `COND_LENGTH` and `STACK_PRM` used for subsections of a Cond where `CalcSub` is called recursively aka within functions, function parameters an jumps.
| Byte | Name                     | Data Type |
| ---- | ------------------------ | --------- |
| 1-2  | DataSize (`COND_LENGTH`) | uint16    |
| 3    | ExtData (`STACK_PRM`)    | int8      |

> Notes: ExtData can be simplified for functions to be the function count as `READ_PARAM` is worth 1 within `STACK_PRM` calculation (although functions are their own subsection, as denoted by the CTYPE and therefore do not affect the `STACK_PRM` calculation for the main Cond) and `02` for integer/float function parameters.

### 6.1 Examples
| CType      | Meaning                                                   |
| ---------- | --------------------------------------------------------- |
| `00 06 02` | Integer.                                                  |
| `00 1C 03` | 3-parameter function                                      |
| `00 13 02` | 2-parameter function                                      |
| `00 0A 01` | 1-parameter function                                      |
| `00 01 00` | 0-parameter function                                      |

## 7. Example Structures

This example demonstrates how to encode a call to the function `SetGlobalBitFlag(hash FlagID, int Value)`. 
The function accepts **two parameters**, both numeric, and is (like all others) invoked using the `READ_FUNCTION` opcode.
First let's start off with our `READ_FUNCTION` (`35`). Immediately following this is the *function identifier*, stored as the CRC-32 (ISO-HDLC) hash of the function name:
```hex
35 <- READ_FUNCTION
18 2B 37 5A <- 0x182B375A is the CRC32 of "SetGlobalBitFlag"
```
After the function hash we have the *CType* (See § 6), which describes the size and other data of the function and its parameters.

Numeric parameters occupy 9 bytes each - including their individual CTypes.

Since this function has two numeric parameters, the total parameter size is `(9 × 2) + 1 = 19 bytes = 0x13`. In this case, the extra `+1` accounts for the CType’s ExtData byte which stores the parameter count when referencing a function.
Each parameter begins with `READ_PARAM`, followed by its CType and value:
```hex
28 <- READ_PARAM
00 06 02 <- CType for a numeric value
34 <- READ_HASH
12 34 56 78 <- 0x12345678 is the example FlagID we will be using
```
Repeating this for the second parameter yields:
```hex
28 <- READ_PARAM
00 06 02 <- CType
32 <- READ_LITERAL
00 00 00 01 <- Value; 0x00000001 = 1
```
Putting everything together, the full byte sequence is:

```hex
35 <- READ_FUNCTION
18 2B 37 5A <- 0x182B375A is the CRC32 of "SetGlobalBitFlag"
00 13 02 <- CType for this function
28 <- READ_PARAM
00 06 02 <- CType for this param
34 <- READ_HASH
12 34 56 78 <- 0x12345678 aka the FlagID
28 <- READ_PARAM
00 06 02 <- Ctype for the second param
32 <- READ_LITERAL
00 00 00 01 <- 0x00000001/1 aka the Value
```

Optionally, assuming this is the entire Cond we can build the header for that sequence to give us:
```hex
00 00 00 <- HEADER data; this is empty space for the engine to fill
00 1B <- COND_LENGTH this is 0x1B because there are 0x1B bytes proceeding this within the cond
02 <- STACK_PRM see § 1.2.2 for the formula
35 <- READ_FUNCTION
18 2B 37 5A <- 0x182B375A is the CRC32 of "SetGlobalBitFlag"
00 13 02 <- CType for this function
28 <- READ_PARAM
00 06 02 <- CType for this param
34 <- READ_HASH
12 34 56 78 <- 0x12345678 aka the FlagID
28 <- READ_PARAM
00 06 02 <- Ctype for the second param
32 <- READ_LITERAL
00 00 00 01 <- 0x00000001/1 aka the Value
```
Or `AAAAABsCNRgrN1oAEwIoAAYCNBI0VngoAAYCMgAAAAE=` when converted to Base64.

TLDR:
* The function hash identifies the target.
* The function CType defines parameter count and total size.
* Each parameter is encoded independently using `READ_PARAM`.

### Examples and Notes
Due to the lack of an array data type functions are often used to emulate this, take for instance the function `0x77B463E5`. It accepts the param(s): `(int: index)` and returns a Boolean output. To be specific this function allows you to input the index of a Psychic Blasters "Stage" and it returns a value representing whether the boss has/hasn't been defeated yet.
CExpression Functions aren't always read only tools used for runtime conditionals - they can be general purpose expressions that mutate values as a side effect - take for instance `RunTrigger` (`0x6984E3AF`). In this function you can pass a hash representing a `TriggerID` and it will run the trigger, returning `1` (`true`) to make sure the conditionals pass.

The naming schema used in CExpression Functions (just like level5 favours in general) can be shown as follows:
* Level5 uses Pascal Case
* A mix of English and Hepburn romanisation.
  * The English is usually not the same localised name's used in game; similar to `cfg.bin`s. Take for instance, Orge ("typo" intentional) instead of Oni.
* Typos, take for instance: `IsApeearMitibiki()`; these are thought to be intentional decisions to avoid collisions due to the level5 engine's extreme reliance on CRC-32 hashes.
* Abbreviations used such as `Util`, `Cnt` etc.
* Common prefixes include `"Get"`, `"Set"`, `"Is"`, `"Common"`, `"Target"`, `"Run"`, `""` and `"Has"`.
* Common suffixes include `"Cnt"`, `"Num"`, `"Now"` and `"Rate"`.

Cond Examples:
`RunTrigger`:
```hex
00 00 00 <-- Header
00 12 <-- COND_LENGTH (0x12 / 18) bytes after this left within the cond
02 <-- STACK_PRM

35 <-- READ_FUNCTION
69 84 E3 AF <-- BE CRC-32 of RunTrigger
00 0A 01 <-- Function CType
28 <-- READ_PARAM
00 06 02 <-- PARAM CType
34 <-- READ_HASH
0E 6B 6F 6B <!-- 0x0E6B6F6B
```
Since `RunTrigger` always returns 1; the cond will always succeed running the trigger `0x0E6B6F6B` as a "byproduct" of the function call (Although the intended purpose of the CExpression)
`GameClear` (Has Main Story been completed):
```hex
00 00 00 <-- HEADER
00 0F <-- COND_LENGTH; this means there are 0xF (15) bytes left in the cond after the COND_LENGTH (including the STACK_PRM)
05 <-- STACK_PRM is 0x5 because there's 1 function, 1 READ_LITERAL and an OPERATOR which is 2(1 * 1) + 1 aka 5 

35 <-- READ_FUNCTION
10 B1 40 96 <-- CRC32 of "GameClear" - aka the function to be executed
00 01 00 <-- Function CType for a function without any parameters

<-- function is ran and it's value is pushed to stack -->

32 <-- READ_LITERAL
00 00 00 01 <-- 0x00000001/1

<-- the 1 is pushed to stack -->

78 <-- OPERATOR: ==
<-- the == pops the 2 values of the stack and pushes the boolean result -->

<-- finally after the cond has been evaluated the value in stack is checked; if truthy the cond suceeded, else it failed -->
```

### 8. History

* IEGO (2011, December)
  * IEGO had a very simplistic Cond system, with only 4 used CExpression Functions, and 7 used Operators (including `8F`).
* Yo-kai Watch 1 (2013, July)
  * Level5 made a lot of changes in this game - which makes sense considering how long they were working on it for - proven by the fact that there are files in Yo-kai Watch 2 that haven't been touched since 2011!
  * Yo-kai Watch 1 for smartphone had over 20 operators, multi-param functions and 118 CExpression functions!
  * Overall this is what truly started the modern Cond system.
* ... future games have not changed the format much aside from adding/removing CExpression Functions

## 9. WIP -- Parsing Flow
This section documents the runtime evaluation flow of a Cond as implemented by:
```cpp
bool yw::util::CExpression::CalcSub(char const* cond, short len)
```
Before the Cond even reaches CalcSub it gets decoded and the `HEADER` (first 3 bytes) are stripped, since there isn't much going on we will only document `CalcSub` as mentioned above:
Unlike what you'd expect, the engine does not perform a full pre-parse of the Cond. Instead, CalcSub validates a minimal header and then executes instructions sequentially while mutating an internal value stack.

### 9.1 Initial Validation
Upon entry, CalcSub receives:
* `cond`: a pointer to the start of a Cond sub-block
* `len`: the number of bytes remaining in the parent context

The function immediately performs a series of important structural checks for safety. If *any* of these checks fail, the Cond will stop being processed any further, returning `false`.
First it performs a minimum length check:
```cpp
if (len < 3)
    return false;
```
A valid Cond by the time it's sent to CalcSub must start with the following 3 bytes at the minimum:
* `COND_LENGTH` (2 bytes)
* `STACK_PRM` (1 byte)
Any shorter buffer will immediately return `false`.
The first two bytes of `cond` are interpreted as a BE uint16:
```cpp
blockLength = (cond[0] << 8) | cond[1];
```
This value represents the number of bytes remaining after the length field itself, including:
* `STACK_PRM`
* all instructions in this Cond
The following conditions immediately invalidate the Cond:
* `COND_LENGTH == 0`
  * At the *absolute minimum* `COND_LENGTH` must be `1` to fit `STACK_PRM`. 
* `len - 2 < COND_LENGTH` (`COND_LENGTH` is shorter than the Cond without itself)

The third byte (`cond[2]`) is read as `STACK_PRM`:
```c
STACK_PRM = cond[2];
if (STACK_PRM == 0)
    return false;
```
A `STACK_PRM` of `0` is considered invalid and causes immediate failure.

### 9.2 Initial well.. Initialisation
Once the header is validated:
* `blockLength` is decremented by 1.
* An instruction pointer is initialized to `cond + 3`.
  * This is right after the `STACK_PRM` aka where the main Cond actually starts.
* A loop counter is derived from `STACK_PRM`.
```cpp
blockLength -= 1;
ptr = cond + 3;
```
Before executing an opcode, the engine performs a range check on the current byte:
```cpp
if (*ptr < 0x28 || *ptr > 0x97)
    return false;
```
### 9.3 OPCODEs
Since opcodes are checked in ascending order, the first opcode is `READ_PARAM (0x28)`:

Once encountered a nested block length is extracted from the two bytes following the `READ_PARAM` (The `DataSize` section of the CTYPE). Then `CalcSub` is called recursively on the parameter sub-block and execution resumes after the sub-block unless either:
  * The parameter count is invalid
  * The declared size is zero

If either condition is met, the engine skips execution of the sub-block and advances the instruction pointer. Since the params are sub-blocks processed by a different `CalcSub` instance that contains the same stack function parameters are evaluated as mostly isolated sub-Cond blocks

<note: finish the rest>

---
<!-- secret easter egg: n123 is coolz (or am I?) -->

# Roadmap
## v1.4
* Has most of the features in the current version (v1.3977 as of now), but with:
  * Improved parsing - removing legacy parsing remnants such as EOCIs and AEOCIs
  * Cond Workshop
    * A subpage which can be used to easily edit, merge and create entirely new Conds!
  * Improved Light Mode 
## v1.5
* This update will mainly focus on making things easier to use; the lib will be easily separable from the UI, there will be more configs etc
  * This will be done in preparation for being built into `yw-mod`, an upcoming project I'm working on!
