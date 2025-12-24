# yw-cond
A web-based work-in-progress library and UI for parsing, decompiling, analyzing and generating Yo-kai Watch conds, with frequent updates.

This parser is for the Yo-kai Watch franchise which has a much more complex cond set (and some slight changes) from Inazuma Eleven: GO, for IEGO cond parsing take a look at the newly released [level5_condition](https://github.com/Tiniifan/level5_condition/) made by Tiniifan themself!

Examples of how the games use conds can be shown below:
* Battle AI
* NPC/Treasure Chest Availability
* Quest/Trophy Conditions
* Shop Item/Price Conditions
* Story Progression
* NPC Interaction handler
* and much, much more!

[The website can be downloaded or viewed here](https://n123git.github.io/yw-cond).

# CExpression (Cond) System Documentation

The **CExpression system**, is a proprietary (usually Base64-encoded) system used for evaluating RPN (Reverse Polish Notation) runtime conditions known as conds. These conds are evaluted by `CExpression::CalcSub` internally and contain *one* or more conditions which consist of:
  * Literal values (Constants, IDs etc)
  * Functions (Engine calls)
  * Operators (arithmetic, logicalal and bitwise)
  * Jumps (conditional and unconditional)
    * These jumps can only move forward and therefore can only be used for if else, NOT complex control flow such as loops. This sadly means the CExpression system is *NOT* turing-complete.
> Note: conditions have been found which contain < 1 conditions but these are invalid and are the reason for the music app being broken in localised versions of Yo-kai Watch 2: Psychic Specters.

> Note: within the following documentation assume all numbers encased in a code block i.e. `02` are in hexadecimal (base-16) unless otherwise specified.
## 1. **Cond Structure**
Each Cond begins with a header section composed of a 3-byte header of `00 00 00`, and a section previously referred to as the `COND_CODE`. This is composed of a uint16 `COND_LENGTH` and a uint8 `STACK_PRM`.
> This header will always be `00 00 00` in the cond itself; the engine will fill it in with the appropriate data during parsing.

### 1.1 Header (3 bytes)
Previously, the header was assumed to always be a **4** byte constant of `00 00 00 00`, with the purpose of serving as an integrity check. But recent developements have shown that the header is composed of a uint16 and uint8 value:
| Byte | Description                                                                                                                                                                                                       |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1-2  | When decoded using a standard decoder; this always appears as `00 00`. Although in level5's decoder they write a uint16 to it equivalent to the amount of bytes in the cond proceeding it.                        |
| 3    | This byte serves an unknown purpose an appears to always be `00`.                                                                                                                                                 |

### 1.2 Cond Code (3 bytes)
a `COND_CODE` is the old name given to a 3-byte (previously though to be 2-byte) section of a cond's header now known as two seperate parts the uint16 `COND_LENGTH` and uint8 `STACK_PRM.` Sections describing the layout of this section can be found below:

#### 1.21 COND_LENGTH
This uint16 defines the total length excluding itself (and all prior bytes) but including all subsequent bytes within the Cond - including the `STACK_PRM`.

#### 1.22 STACK_PRM
This byte known as `STACK_PRM` is equal to the sum of the quantity of top-level values multiplied by 2 and the amount of operators; it can be simplified to `(((READ_FUNC_CNT + LIT_NONPARAM_CNT) * 2) + OP_CNT)` where `LIT_NONPARAM_CNT` is the amount of `READ_LITERAL` and `READ_HASH`(s) that aren't used as function parameters.

> Example: `00 00 00 - 00 0F - 05 - 35 10 B1 40 96 00 01 00 32 00 00 00 01 78`

---

## 2. **Data Identifiers**

### 2.1 Reads

| Type            | Hex Code | Decimal Code | Description                                                                                     |
| --------------- | -------- | ------------ | ----------------------------------------------------------------------------------------------- |
| READ_PARAM      | `28`     | `40`         | Reads a param withan a `READ_FUNCTION` call. Followed by a CTYPE.                               |
| READ_LITERAL    | `32`     | `50`         | Pushes an integer of type 4.                                                                    |
| READ_FLOAT      | `33`     | `51`         | Pushes an IEEE 754 float of type 6.                                                             |
| READ_HASH       | `34`     | `52`         | Similar to a READ_LITERAL but is used to push a hash instead. Internally functions identically. |
| READ_FUNCTION   | `35`     | `53`         | Reads and pushes a function. Followed by a CTYPE representing the function as a whole.          |

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

Operators are grouped by the multiple of ten in their decimal opcode. Each `x0–x9` range represents a distinct operator class, and operators within a range are closely related in behavior as shown by the table below:

| Range | Category                | Operators Included      |
| ----- | ----------------------- | ----------------------- |
| `7X`  | Unary Arithmetic        | `++`, `--`              |
| `8X`  | Unary Bitwise / Logical | `~`, `!!`               |
| `9X`  | Binary Arithmetic       | `*`, `/`, `%`, `+`, `-` |
| `1X`  | Bit Shifting            | `<<`, `>>`              |
| `11X` | Relational Comparison   | `<`, `<=`, `>`, `>=`    |
| `12X` | Equality Comparison     | `==`, `!=`              |
| `13X` | Bitwise Binary Logic    | `&`, `\|\|`, `^`        |
| `14X` | Logical Binary Logic    | `&&`, `\|`              |

## 3.1. Jumps

There are two kinds of jumps supported by the CExpression engine, these can be considered as psuedo-ops:
| Hex Code | Decimal Code | Symbol  | Operation                             | Notes                                                                                                  |
| -------- | ------------ | ------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `96`     | `150`        | ?->     | Conditional Jump Operator             | Jumps forward Y bytes if X is falsy. Unofficial symbol.                                                |
| `97`     | `151`        | ->      | Unconditional Jump Operator           | Jumps forward X bytes and optionally spawns another `CalcSub` instance for the gap. Unofficial symbol. |

Jumps are followed by a uint16 `JUMP_CNT` value which decides how many bytes to jump *from* the end of the `JUMP_CNT` - if misaligned, this can cause the evaluator to throw so be very careful when implementing jumps manually. Additionally, as of writing this `yw-cond` dosen't support these in decompilation, recompilation or parsing so you'll have to manually test them.
Despite being confirmed to have existed as early as Yo-kai Watch 1, these jumps are rarely used in practice. Although these can most likely be used to get around the 64 stack value limit.

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

> Note: Like all values within conds, Literal Values are *big-endian*.
<!-- Optional `READ_PARAM` chains  <!-- for a bit this was chins 😭 --> allow for unlimited nesting of subsections for multi-param functions. -->

### 4.2 READ_LITERAL
* Always followed by a *signed 32-bit integer* (but can also be used for functions that expect a 16-bit, 8-bit, or boolean value, in which case it should still padded to 4 bytes).
* Values are **big-endian**.
* Pushes an integer of type 4 (int32)

### 4.3 READ_FLOAT
* Pushes an IEEE-754 (standard float32) float of type 6 (float32)
* Very rare.

### 4.4 READ_HASH
* Pushes an integer of type 4 (int32)
  * Used for IDs hence the name `READ_HASH`
* Functionally the same as `READ_LITERAL`; atleast in `CExpression::CallSub` for whatever reason.

## 5. CExpression Data Types
These data types are *never directly referenced in the cond itself* but are used internally by the CExpression engine during evaluation and will therefore be mentioned here because they are important to understand the engine itself.
| ID | Type   | Description             |
| -- | ------ | ----------------------- |
| 0  | int8   | 8-bit signed integer    |
| 1  | uint8  | 8-bit unsigned integer  |
| 2  | int16  | 16-bit signed integer   |
| 3  | uint16 | 16-bit unsigned integer |
| 4  | int32  | 32-bit signed integer   |
| 5  | uint32 | 32-bit unsigned integer |
| 6  | float  | 32-bit floating point   |

## 6. **CTypes**
CTypes are **3-byte descriptors** used to represent properties of functions and function parameters.
| Byte | Purpose                |
| ---- | ---------------------- |
| 1-2  | DataSize               |
| 3    | ExtData                |

Notes:
* The `DataSize` represents the length (in bytes) of the value from the ExtData onwards
* For functions, ExtData represents the number of parameters.
* For integers, the purpose of ExtData is unknown although it is always `02`.

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
The function takes **two parameters**, both numeric, and is (like all others) invoked using the `READ_FUNCTION` opcode.
First let's start off with our `READ_FUNCTION` (`35`). Immediately following this is the *function identifier*, stored as the CRC-32 (ISO-HDLC) hash of the function name:
```hex
35 <- READ_FUNCTION
18 2B 37 5A <- 0x182B375A is the CRC32 of "SetGlobalBitFlag"
```
After the function hash we have the *CType* (See § 13), which describes the size and other data of the function and its parameters.

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

Optionally, assuming this is the entire cond we can build the header for that sequence to give us:
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
* `READ_FUNCTION` invokes the call.
* The function hash identifies the target.
* The function CType defines parameter count and total size.
* Each parameter is encoded independently using `READ_PARAM`.

### Examples and Notes
Due to the lack of an array data type functions are often used to emulate this, take for instance the function `0x77B463E5`. It takes the param(s): `(int: index)` and returns a boolean output. To be specific this function allows you to input the index of a Psychic Blasters "Stage" and it return a value representing whether the boss has/hasn't been defeated yet.
CExpression Functions aren't always read only tools used for runtime conditionals - they can be general purpose expressions that mutate values as a side effect - take for instance `RunTrigger` (`0x6984E3AF`). In this function you can pass a hash representing a `TriggerID` and it will run the trigger, returning `1` (`true`) to make sure the conditionals pass.

The naming schme used in CExpression Functions (just like level5 favours in general) can be shown as follows:
* Level5 uses Pascal Case
* A mix of english and hepburn romanisation.
  * The english is usually not the same localised name's used in game; similar to `cfg.bin`s. Take for instance, Orge instead of Oni.
* Typos, take for instance: `IsApeearMitibiki()`; these are thought to be intentional decisions to avoid collisions due to the level5 engine's extreme reliance on CRC-32 hashes.
* Abbreviations used such as `Util`, `Cnt` etc.
* Common prefixes include `"Get"`, `"Set"`, `"Is"`, `"Common"`, `"Target"`, `"Run"`, `""` and `"Has"`.
* Common suffixes include `"Cnt"`, `"Num"`, `"Now"` and `"Rate"`.

Cond Examples:
* Single Condition: RunTrigger:
  * `00 00 00 00 12 02 35 69 84 E3 AF 00 0A 01 28 00 06 02 34 0E 6B 6F 6B`
  * This can be parsed as:
    * `00 00 00` - Header.
    * `00 12` - COND_LENGTH shows that after it there will be 0x12 bytes remaining.
    * `02` - STACK_PRM.
    * `35` - READ_FUNCTION; tells the game what layout to expect and to read a value.
    * `69 84 E3 AF` - The CRC32 hash of `RunTrigger`.
    * `00 0A 01` - The CTYPE for a function that takes one parameter aka `RunTrigger` in this example.
    * `28` - Used to tell the engine to keep reading so it correctly parses the params.
    * `00 06 02` - The CTYPE for an integer; referring to the input of `RunTrigger` aka the `TriggerID`
    * `34`- Similar purpose as `28`.
    * `0E 6B 6F 6B` - Input of the function (in this case it's the `TriggerID`); pushes the function and params to the stack. 
    * No OPERATORs or other values; so it runs the above function and checks if the output is truthy; as this function is a `Set` not a `Get/Is/Has` function it always returns `1` (`true`); a *truthy* value.
* Single Condition: GameClear (Has Main Story been completed):
  * `00 00 00 00 0F 05 35 10 B1 40 96 00 01 00 32 00 00 00 01 78`
  * This can be parsed as:
    * `00 00 00` - Header
    * `00 0F 05` - Condcode.
    * `35` - READ_FUNCTION; tells the game what layout to expect and to read a value.
    * `10 B1 40 96` - The CRC32 hash of `GameClear`.
    * `00 01 00` - The CType for a Zero-Param Function; telling the engine to not parse any params, pushing the function's output as a value to stack.
    * `32` - A READ_LITERAL telling the parser to expect a literal value next.
    * `00 00 00 01` - A literal value equivalent to `1` (`true`) which promptly gets pushed to stack.
    * `78` - an OPERATOR; specifically `==`; it compares both values in stack and returns a boolean output deciding whether the cond fails or succeeds.

---

### 8. History

* IEGO (2011, December)
  * IEGO had a very simplistic cond system, with only 4 used CExpression Functions, (3 used CTYPEs?) and 7 used Operators (including `8F`).
* Yo-kai Watch 1 (2013, July)
  * Level5 made alot of changes in this game - which makes sense considering how long they were working on it for; this can be proven by the files in Yo-kai Watch 2 that haven't been touched since 2011!
  * Yo-kai Watch 1 for smartphone had over 20 operators, multi-param functions and 118 CExpression functions!
  * Overall this is what truly started the modern cond system.
* ... future game's have not changed the format much aside from adding/removing CExpression Functions

## 9. W.I.P -- General Parsing Flow

<>

---
<!-- secret easter egg: n123 is coolz (or am I?) -->

# Roadmap
## v1.4
* Has most of the features in the current version (v1.3977 as of now), but with:
  * Improved parsing - removing legacy parsing remnants such as EOCIs and AEOCIs
  * Cond Workshop
    * A subpage which can be used to easily edit, merge and create entirely new conds!
  * Improved Light Mode 
## v1.5
* This update will mainly focus on making things easier to use; the lib will be easily separable from the UI, there will be more configs etc
  * This will be done in preperation for being built into `yw-mod`, an upcoming project I'm working on!
