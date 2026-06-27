# CExpression Format Specification

The CExpression system is a proprietary, stack-based, big-endian, recursive Reverse Polish Notation (RPN) binary format used to evaluate runtime conditions in Level5's engines from IEGO onwards. Colloquially referred to as "Conds" (historically "PhaseAppear"), they are evaluated by the internal `yw::util::CExpression::CalcSub` function. For the usage of CExpressions in the Yo-kai Watch series, please refer to the project README, for usage of `yw-cond`, refer to the usage docs. This documentation assumes you have pre-requisite knowledge on binary formats.

Because jumps can only move forward, the system can express `if`/`if-else` logic but is **not Turing-complete** (loops are impossible).

> Note: Invalid CExpressions evaluate to `false`. This is the reason the music app is broken in localized versions of *Yo-kai Watch 2: Psychic Specters*. Additionally, within this specification, numbers enclosed in code blocks (e.g., `02`) are in hexadecimal format unless otherwise specified.

## 1. Binary Structure

Every CExpression begins with a 6-byte header, followed by the instruction payload.

### 1.1 Header (3 bytes)
The first 3 bytes of a Cond are reserved. In the raw state, this must be `00 00 00`.

### 1.2 Root CType (3 bytes)
Immediately following the header is a 3-byte block. This block is really just a root `CType` (see Section 2), however for simplicity one may treat it differently.

* `COND_LENGTH` (`uint16`): Defines the total length of the remaining bytes in this Cond block, including the `STACK_PRM` byte itself, but excluding the `COND_LENGTH` bytes.
* `STACK_PRM` (`int8`): Specifies the quantity of elements in the current recursed subsection. 

## 2. CTypes (3 bytes)

CTypes are 3-byte structural descriptors used to define the recursive subsections, the CExpression system relies on (take for instance, function parameters or jump bodies). Because of said recursive implementation, CTypes allow passing complete nested expressions as parameters.

| Offset | Size | Field | Type | Description |
| :--- | :--- | :--- | :--- | :--- |
| 0 | 2 | `Length` | `uint16` | Length of the subsection in bytes (excluding these 2 bytes). |
| 2 | 1 | `STACK_PRM` | `int8` | The `STACK_PRM` for this specific subsection. |

> Note: CTypes contribute to the `Length` of their parent block, but they **do NOT** increment the parent's `STACK_PRM`, nor do the CType's contents (however they do contribute to the `Length`). Every element except CTypes contributes to their direct parent CType's `STACK_PRM`.

## 3. Opcodes

All instructions fall between `0x28` and `0x97`. The stack has a hard limit of 64 values.

### 3.1 Read Operations

These instructions push values onto the stack. 

| Value | Name | STACK_PRM Contribution | Description |
| :--- | :--- | :--- | :--- |
| `0x28 / 40` | `READ_PARAM` | +1 | Reads a parameter within a `READ_FUNCTION` call. Followed by a `CType` defining the parameter's sub-block. |
| `0x32 / 50` | `READ_LITERAL` | +2 | Pushes a signed 32-bit integer (`CValue` type 4). The opcode contributes +1, and the accompanying 4-byte literal value contributes +1. Value is Big-Endian and padded even if the target function expects a smaller type (e.g., bool, int8). |
| `0x33 / 51` | `READ_FLOAT` | +2 | Pushes an IEEE 754 single-precision float (`CValue` type 6). The opcode contributes +1, and the accompanying 4-byte float value contributes +1. Extremely rare pre Yo-kai Watch 2. |
| `0x34 / 52` | `READ_HASH` | +2 | Pushes a signed 32-bit integer (`CValue` type 4). The opcode contributes +1, and the accompanying 4-byte hash value contributes +1. Functionally identical to `READ_LITERAL`, but semantically used for IDs (e.g., FlagIDs, BaseIDs). |

### 3.2 Function Calls

| Value | Name | STACK_PRM Contribution | Description |
| :--- | :--- | :--- | :--- |
| `0x35 / 53` | `READ_FUNCTION` | +2 | Executes a CExpression function and pushes the result. The opcode contributes +1, and the accompanying 4-byte CRC-32 hash contributes +1. |

Structure:
```text
35               ; READ_FUNCTION Opcode
[4 bytes]        ; CRC-32 (ISO-HDLC) hash of the function name
[CType]          ; Defines the Length and STACK_PRM of the function call
[PARAMS...]      ; Zero or more READ_PARAM blocks
```

### 3.3 Operators

Operators pop values from the stack (LIFO), perform an operation, and push the result. They are grouped logically by their decimal ranges. All operators contribute +1 to their parent's `STACK_PRM`.
> Note that operators employ implicit type promotion so, for instance, adding a `uint8` and a `float` will produce a `float`.

#### Unary Operators
| Value | Symbol | STACK_PRM Contribution | Operation |
| :--- | :--- | :--- | :--- |
| `0x46 / 70` | `++` | +1 | Non-mutating increment (pops value, adds 1, pushes result). |
| `0x47 / 71` | `--` | +1 | Non-mutating decrement (pops value, subtracts 1, pushes result). |
| `0x50 / 80` | `~` | +1 | Bitwise NOT |
| `0x51 / 81` | `(bool)` | +1 | Boolean Cast (Returns `1` if != 0, else `0`) |

#### Binary Arithmetic
| Value | Symbol | STACK_PRM Contribution | Operation |
| :--- | :--- | :--- | :--- |
| `0x5A / 90` | `*` | +1 | Multiply |
| `0x5B / 91` | `/` | +1 | Divide |
| `0x5C / 92` | `%` | +1 | Modulus |
| `0x5D / 93` | `+` | +1 | Add |
| `0x5E / 94` | `-` | +1 | Subtract |

#### Bitwise & Shift
| Value | Symbol | STACK_PRM Contribution | Operation |
| :--- | :--- | :--- | :--- |
| `0x64 / 100` | `<<` | +1 | Left Shift |
| `0x65 / 101` | `>>` | +1 | Right Shift |
| `0x82 / 130` | `&` | +1 | Bitwise AND |
| `0x83 / 131` | `\|` | +1 | Bitwise OR |
| `0x84 / 132` | `^` | +1 | Bitwise XOR |

#### Comparison
| Value | Symbol | STACK_PRM Contribution | Operation |
| :--- | :--- | :--- | :--- |
| `0x6E / 110` | `<` | +1 | Less Than |
| `0x6F / 111` | `<=` | +1 | Less or Equal |
| `0x70 / 112` | `>` | +1 | Greater Than |
| `0x71 / 113` | `>=` | +1 | Greater or Equal |
| `0x78 / 120` | `==` | +1 | Equal |
| `0x79 / 121` | `!=` | +1 | Not Equal |

#### Logical
| Value | Symbol | STACK_PRM Contribution | Operation |
| :--- | :--- | :--- | :--- |
| `0x8F / 143` | `&&` | +1 | Logical AND (Most common operator) |
| `0x90 / 144` | `\|\|` | +1 | Logical OR |

### 3.4 Control Flow (Jumps)

Jumps are forward-only pseudo-ops used for conditional execution. They are immediately followed by a `CType` which dictates the size of the jump body and execution behavior via its `Length` and `STACK_PRM`. Both jumps contribute +1 to their parent's `STACK_PRM`, as you'd expect.

| Value | Symbol | STACK_PRM Contribution | Description |
| :--- | :--- | :--- | :--- |
| `0x96 / 150` | `?->` | +1 | Conditional Jump. Pops a value. If the value is truthy (!= 0) AND the CType's `STACK_PRM` > 0, the sub-block is executed then skipped. Otherwise, it is skipped entirely. |
| `0x97 / 151` | `->` | +1 | Unconditional Jump. If the CType's `STACK_PRM` > 0, the sub-block is executed then skipped. Otherwise, it is skipped. |

## 4. Internal Data Types (CValues)

CValues exist purely *at runtime* on the stack; they are not explicitly defined in this format.

| ID | Type | Description |
| :--- | :--- | :--- |
| 0 | `int8` | 8-bit signed integer |
| 1 | `uint8` | 8-bit unsigned integer |
| 2 | `int16` | 16-bit signed integer |
| 3 | `uint16` | 16-bit unsigned integer |
| 4 | `int32` | 32-bit signed integer (Default) |
| 5 | `uint32` | 32-bit unsigned integer |
| 6 | `float` | 32-bit IEEE 754 float |

> Aside from `int32` and `float`, these other types can only be obtained through function calls and type promotion, descending from a function call.

## 5. Examples

### 5.1 Function Call: `SetGlobalBitFlag(FlagID, Value)`
This function sets a global bit flag. It requires two numeric parameters.
*   CRC-32 of `"SetGlobalBitFlag"` = `0x182B375A`
*   Parameter 1 (FlagID): `0x12345678` (using `READ_HASH`)
*   Parameter 2 (Value): `0x00000001` (using `READ_LITERAL`)
*   Param size: 9 bytes * 2 params = 18 bytes. Add 1 for the function CType's `STACK_PRM` byte = 19 (`0x13`).

**Raw Hex:**
```hex
00 00 00       <- Header
00 1B          <- COND_LENGTH (27 bytes remaining)
02             <- STACK_PRM (2 elements: the function opcode, and the function hash)
35             <- READ_FUNCTION
18 2B 37 5A    <- CRC-32 Hash
00 13 02       <- CType: Length 19, STACK_PRM 2 (two params)
  28           <- READ_PARAM
  00 06 02     <- Param CType: Length 6, STACK_PRM 2 (read op + value)
  34           <- READ_HASH
  12 34 56 78  <- FlagID
  28           <- READ_PARAM
  00 06 02     <- Param CType: Length 6, STACK_PRM 2
  32           <- READ_LITERAL
  00 00 00 01  <- Value (1)
 ; we then end up with SetGlobalBitFlag(0x12345678, 1)
```
In Base64, this would be `AAAAABsCNRgrN1oAEwIoAAYCNBI0VngoAAYCMgAAAAE=`.

### 5.2 Condition Check: `GameClear() == 1`
Checks if the main story is completed.
* CRC-32 of `"GameClear"` = `0x10B14096`

```hex
00 00 00       <- Header
00 0F          <- COND_LENGTH (15 bytes)
05             <- STACK_PRM (5: Function opcode, Function hash, Literal opcode, Literal value, Operator)
35             <- READ_FUNCTION
10 B1 40 96    <- CRC-32 Hash
00 01 00       <- CType: Length 1, STACK_PRM 0 (no params)
32             <- READ_LITERAL
00 00 00 01    <- Value (1)
78           <- OPERATOR: ==
```

<!--## 6. Runtime Evaluation Flow (`CalcSub`)

CExpressions are evaluated via `yw::util::CExpression::CalcSub(char const* cond, short len)`. The engine does *not* build an AST; it executes sequentially, mutating an internal value stack.

### 7.1 Initialization & Validation
Upon entry, `CalcSub` receives a pointer to a sub-block and its remaining length. It performs strict validation. Failure at any point returns `false`.
1.  **Length Check:** `len` must be $\ge 3$.
2.  **Parse Header:** Read `COND_LENGTH` (`uint16` BE) and `STACK_PRM` (`int8`).
3.  **State Validation:**
    *   `COND_LENGTH` cannot be `0` (must at least fit the `STACK_PRM` byte).
    *   `len - 2` must be $\ge$ `COND_LENGTH`.
    *   `STACK_PRM` cannot be `0`.
4.  **Setup:** Decrement `blockLength` by 1, set instruction pointer `ptr = cond + 3`.

### 7.2 Execution Loop
The engine iterates based on `STACK_PRM`. Before executing, it verifies the opcode is within the valid range: `0x28 <= *ptr <= 0x97`.

*   **`0x28` (READ_PARAM):** Reads the subsequent `CType` to get the parameter block `Length`. Calls `CalcSub` recursively on this sub-block. If the parameter size is 0 or invalid, it skips execution and advances the pointer.
*   **`0x32` - `0x34` (READ_LITERAL/FLOAT/HASH):** Advances the pointer by 5 bytes (1 opcode + 4 value bytes) and pushes the value to the stack.
*   **`0x35` (READ_FUNCTION):** Reads the 4-byte hash, resolves the function pointer, reads its `CType`, recursively processes any `READ_PARAM` blocks attached to it, and invokes the engine syscall. The return value is pushed to the stack.
*   **`0x46` - `0x90` (Operators):** Pops the required operands (1 or 2), performs the math/logic, and pushes the result.
*   **`0x96` - `0x97` (Jumps):** Reads the corresponding `CType`. Evaluates the condition (for `0x96`) and the `STACK_PRM` byte. If execution is required, calls `CalcSub` recursively on the jump body. Finally, advances the pointer past the jump body.

Once the loop completes, the final value left on the stack determines the success or failure of the Cond.
-->

## 6. Historical Context

* IEGO (Dec 2011): Utilised a very simple subset of the system, utilizing only 4 known functions and 7 operators.
* Yo-kai Watch 1 (Jul 2013): Ushered in the modern CExpression format. The smartphone port featured over 20 operators, multi-param function support, and roughly 118 documented functions.
* In later titles, the binary format and opcode structure have remained largely static. Changes between titles (YW2, YWB, YW3) seem to be restricted exclusively to the addition, removal, or modification of the underlying CExpression functions, not the RPN evaluator itself.

<!-- secret easter egg: n123 is not coolz (or am I? No, no I'm not) -->
