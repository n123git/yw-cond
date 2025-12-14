# yw-cond
A web-based work-in-progress library and UI for parsing, decompiling, analyzing and generating Yo-kai Watch conds, with frequent updates.

This parser is for the Yo-kai Watch franchise which has a much more complex cond set (and some slight changes) from Inazuma Eleven, for IEGO cond parsing take a look at the newly released [level5_condition](https://github.com/Tiniifan/level5_condition/) made by Tiniifan themself!

Examples of how the games use conds can be shown below:
* Battle AI
* NPC/Treasure Chest Availability
* Quest/Trophy Conditions
* Shop Item/Price Conditions
* Story Progression
* and much, much more!


[The website can be downloaded or viewed here](https://n123git.github.io/yw-cond).

# CExpression (Cond) System Documentation

The **CExpression**, or **Cond system**, is a proprietary (usually Base64-encoded) system used for evaluating runtime conditions using RPN (Reverse Polish Notation); it contains *zero* or more conditions which consist of:
  * Literal values (Constants, IDs etc)
  * Functions (Engine calls)
  * Operators (arithmetic, logical (sometimes used as structural), bitwise, and some control flow)

## 1. **Cond Structure**

Each Cond begins with a header section composed of a **3-byte header** and a **3-byte COND_CODE**.
> Note: This is *PER-COND as a whole, NOT* per condition.

### 1.1 Header (3 bytes)
Previously, the header was assumed to always be a **4** byte constant of `00 00 00 00`, with the purpose of serving as an integrity check. But recent developements have shown that the header is composed of a uint16 and uint8 value:
| Byte | Description                                                                                                                                                                                                                                                       |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1-2  | When decoded using a standard decoder; this always appears as `00 00`. Although in level5's decoder they write a uint16 to it equivalent to the amount of bytes in the cond proceeding it.                                                                        |
| 3    | This byte serves an unknown purpose an appears to always be `00`.                                                                                                                                                                                                 |

### 1.2 Cond Code (3 bytes)
a `COND_CODE` is the old name given to a 3-byte (previously though to be 2-byte) section of a cond's header now known as two seperate parts the uint16 COND_LENGTH and uint8 STACK_PRM. Sections describing the layout of this section can be found below:

#### 1.21 COND_LENGTH
This uint16 defines the total length excluding itself (and all prior bytes) but including all subsequent bytes within the Cond - including the `STACK_PRM`.

#### 1.22 STACK_PRM
This byte known as `STACK_PRM` is equal to the amount of top-level values multiplied by 2 combined with the amount of operators; it can be simplified to `(((READ_MEM_CNT + LIT_NONPARAM_CNT) * 2) + OP_CNT)` where `LIT_NONPARAM_CNT` is the amount of `READ_LITERAL`s that aren't used as params in a function.

> Example: `00 00 00 - 00 0F - 05 - 35 10 B1 40 96 00 01 00 32 00 00 00 01 78`

---

## 2. **Data Identifiers**

### 2.1 Reads

| Type            | Hex Code | Description                       |
| --------------- | -------- | --------------------------------- |
| READ_MEMORY     | `35`     | Reads a memory resource           |
| READ_LITERAL    | `32`     | Pushes a literal value (constant) |
| READ_SUBSECTION | `34`     | Reads a subsection of memory/data |

### 2.2 Special Markers

| Type            | Hex Code | Description                                                                                        |
| --------------- | -------- | -------------------------------------------------------------------------------------------------- |
| EXTENSION_DELIM | `28`     | Marker signaling to continue consuming params; followed by a `CTYPE` and then a `READ_SUBSECTION`. |

---

## 3. **Operators**

Operators perform logical, arithmetic, bitwise or control flow operations on one or more values.
| Hex Code | Symbol | Num    | Operation                             | Notes                                                                                                  |
| -------- | ------ | ------ | ------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `46`     | ++     | 1      | Incrementation                        |                                                                                                        |
| `47`     | --     | 2      | Decrementation                        |                                                                                                        |
| `50`     | ~      | 3      | Bitwise NOT                           |                                                                                                        |
| `51`     | !      | 4      | Logical NOT                           |                                                                                                        |
| `5A`     | *      | 5      | Multiply                              |                                                                                                        |
| `5B`     | /      | 6      | Divide                                |                                                                                                        |
| `5C`     | %      | 7      | Modulus                               |                                                                                                        |
| `5D`     | +      | 8      | Addition                              |                                                                                                        |
| `5E`     | -      | 9      | Subtraction                           |                                                                                                        |
| `64`     | <<     | 10     | Left shift                            |                                                                                                        |
| `65`     | >>     | 11     | Right shift                           |                                                                                                        |
| `6E`     | <      | 12     | Less than                             |                                                                                                        |
| `6F`     | <=     | 13     | Less or equal                         |                                                                                                        |
| `70`     | >      | 14     | Greater than                          |                                                                                                        |
| `71`     | >=     | 15     | Greater or equal                      |                                                                                                        |
| `78`     | ==     | 16     | Equal                                 |                                                                                                        |
| `79`     | !=     | 17     | Not equal                             |                                                                                                        |
| `82`     | &      | 18     | Bitwise AND                           |                                                                                                        |
| `83`     | \|     | 19     | Bitwise OR                            |                                                                                                        |
| `84`     | ^      | 20     | Bitwise XOR                           |                                                                                                        |
| `8F`     | &&     | 21     | Logical AND                           | By far the most common.                                                                                |
| `90`     | \|\|   | 22     | Logical OR                            | Frequently combined with `8F`.                                                                         |
| `96`     | ?->    | 23     | Conditional Jump Operator (pseudo-op) | Jumps forward Y bytes if X is falsy. Unofficial symbol.                                                |
| `97`     | ->     | 24     | Unconditional Jump Operator           | Jumps forward X bytes and optionally spawns another `CalcSub` instance for the gap. Unofficial symbol. |

Operators can be grouped by their *high nibble* (first hex digit), and further subdivided by whether the *low nibble* falls within 0–9 (normal position) or A–F (extended/ext position). Here is the grouping:

| High Nibble | Group Name                   | Notes                                         | 
| ----------- | ---------------------------- | --------------------------------------------- | 
| `4`         | Stepper Ops                  | Includes increment and decrement operators.   | 
| `5`         | Unary Ops                    | `4X` is also unary but only covers ++/--.     | 
| `5` (ext.)  | Arithmetic Ops               | Standard arithmetic operations.               |
| `6`         | Bit Shift Ops                | Handles left/right bit shifting.              |
| `6` (ext.)  | Lower Comparison Ops         | Contains `<` and `<=`.                        |
| `7`         | Comparison Ops               | Holds `>` and `>=` operators.                 |
| `8`         | Bitwise Logic Ops            | Handles bitwise AND/OR/XOR.                   |
| `8` (ext.)  | Logical AND Ops              | Contains `&&`.                                |
| `9`         | Logical OR / Conditional Ops | Contains `\|\|`, `?->` and `->`.              |

---


## 4. **Read Operations**

### 4.1 READ_MEMORY

A READ_MEMORY entry is structured as:

```
READ_MEMORY
  LITERAL_VALUE (4 bytes)
  CTYPE (3 bytes; type of the above value)
  optional:EXTENSION_DELIM (1 byte)
     CTYPE (3 bytes; type of the below value)
     READ_SUBSECTION (1 byte)
     LITERAL_VALUE
  ... can recursively hold another CTYPE then EXTENSION_DELIM and so on
```

* The **4-byte literal value** is the value (for integers; although usually still an ID) or CRC32 ISO-HDLC hash of the function name (for functions).
  * Literal Values are *big-endian*.
* Optional **EXTENSION_DELIM+READ_SUBSECTION** chains  <!-- for a bit this was chins 😭 --> allow unlimited nesting of subsections for multi-param functions.

### 4.2 READ_LITERAL

* Always followed by a **32-bit integer** (but may also be used for 16-bit, 8-bit, or boolean values, in which case it will still be padded to 4 bytes).
* Values are **big-endian**.

### 4.3 READ_SUBSECTION

* Always follows an EXTENSION_DELIM/CTYPE combination.
* Represents a smaller section of memory or another value block.

## 5. **Internal Data Types**

These data types are **never directly referenced in the CExpression itself** but are used internally by the engine during execution and will therefore be mentioned here cuz why not :P

| ID | Type   | Description             |
| -- | ------ | ----------------------- |
| 0  | int8   | 8-bit signed integer    |
| 1  | uint8  | 8-bit unsigned integer  |
| 2  | int16  | 16-bit signed integer   |
| 3  | uint16 | 16-bit unsigned integer |
| 4  | int32  | 32-bit signed integer   |
| 5  | uint32 | 32-bit unsigned integer |
| 6  | float  | 32-bit floating point   |
| 7+ | unk    | Currently unknown.      |

---

## 6. **CTypes (True Data Types)**

CTypes are **3-byte descriptors** representing data types used in the engine.

| Byte | Purpose                |
| ---- | ---------------------- |
| 1-2  | DataSize               |
| 3    | ExtData                |

Notes:
* The `DataSize` represents the length (in bytes) of the value from the ExtData onwards
* For functions, ExtData represents the number of parameters.
* For integers, the purpose of ExtData is unknown although it is usually `02`.

### 6.1 CType Examples

| CType      | Meaning                                                   |
| ---------- | --------------------------------------------------------- |
| `00 06 02` | Integer.                                                  |
| `00 1C 03` | 3-parameter function                                      |
| `00 13 02` | 2-parameter function                                      |
| `00 0A 01` | 1-parameter function                                      |
| `00 01 00` | 0-parameter function                                      |

---

### 7. Examples and Notes
Due to the lack of an array data type functions are often used to emulate this, take for instance the function `0x77B463E5`. It takes the param(s): `(int: index)` and returns a boolean output. Specifically, you input a Psychic Blasters Stage Index and it tells you if that boss has/hasn't been defeated yet.

Due to the lack of a string or enum data type, CRC-32 ISO-HDLC hashes are often used instead with lookup table functions.
CExpression Functions aren't always read only - take for instance `RunTrigger()` (0x6984E3AF), it takes in a big endian ID, runs the trigger and always returns `1` (`true`).

The Naming Schema of CExpression Functions can be shown as follows:
* Always Pascal Case; unlike normal IDs where level5 uses a mix of lowercase and snakecase.
* A mix of english and hepburn romanisation.
  * The english is usually not the same localised name's used in game; similar to `cfg.bin`s. Take for instance, Ogre instead of Oni.
* Typos, take for instance: `IsApeearMitibiki()`; these are thought to be intentional decisions to avoid collisions due to the level5 3DS engine's extreme reliance on CRC-32 (ISO-HLDC).
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
    * `35` - READ_MEMORY; tells the game what layout to expect and to read a value.
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
    * `35` - READ_MEMORY; tells the game what layout to expect and to read a value.
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

1. The system begins by reading and validating the HEADER and CONDCODE.
2. Then it reads the next byte i.e. if its a READ_LITERAL, OPERATOR etc
3. Then it keeps going until it finds values to push to stack
2. Operators are always binary (take 2 inputs) and return an output.
3. Functions are the only arbitrary size data type due to the them having up to 255 parameters (although in practice more than 3 has not been witnessed) which can all be functions which contain functions and so on.
4. Logical evaluation uses `&&` (`8F`) and `||` (`90`) markers.
5. The engine resolves the final result based on literals, function returns, and operator results.

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
