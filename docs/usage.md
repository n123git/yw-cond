# Usage Guide
This section will explain how to use the web UI of `yw-cond`. First, let's open the website - you can open the website [by clicking this link](https://n123git.github.io/yw-cond). You should see something like this:
![Blank yw-cond site](../tutorial/tutorial_blankslate.png)
Next, we'll load a random sample CExpression (or 'Cond') to demonstrate, we can do this by clicking *Load Sample* (*Charger un échantillon* in French). This loads a sample from a pre-defined sample list, shuffling in order (mostly used for testing):
![Blank yw-cond site](../tutorial/tutorial_sampleloaded.png)
The $\color{#E81B1B}{■}$ (red) section is where you paste in your CExpression, it can be in base64 i.e. `AAAAABgFNSo9RUMACgEoAAYCNAASNFYyAAAAAXg=` or in hex as shown by the image.
The $\color{#3D00BA}{■}$ (purple) section will display errors if they appear. These should never occur in practice however.
The $\color{#FFAB00}{■}$ (orange) section displays a C/C++ decompilation of the CExpression to show the behaviour of the CExpression.
The $\color{#27F200}{■}$ (green) section displays a 1:1 breakdown of all the bytes within the CExpression and their purpose - this is for people who understand the format - it can also be used for debugging.
Next, let's go over the buttons on the sidebar:
* *Parse* (*Analyser* in French) parses and analyses the cond in the textarea ($\color{#E81B1B}{■}$ (red) section)
  * *Load Sample* (*Charger un échantillon* in French) automatically runs this, same for *Toggle Format* (*Changer de format*)
  * Clicking *Parse* when the textarea is empty will clear the $\color{#FFAB00}{■}$ (orange), $\color{#27F200}{■}$ (green) and $\color{#3D00BA}{■}$ (purple) sections.
* *Toggle Format* (*Changer de format* in French) converts the CExpression in the textarea ($\color{#E81B1B}{■}$ (red) section) from base64 -> hex and vice versa. It also automatically runs the *Parse* (*Analyser*) button.
* *Load Sample* (*Charger un échantillon* in French) loads a sample from a pre-defined sample list, shuffling in order (mostly used for testing). It also automatically clicks the *Parse* (*Analyser* in French) button.
* *Generate Cond Template(s)* (*Générer modèles Cond* in French) opens a modal where you can select several predefined CExpressions i.e. Watch Rank, Weather etc
* *Config* (*Config* in French) allows you to configure several options which range from decompiler changes, to aesthetic changes to debug mode. These are saved to `localStorage` automatically. Each option explains it's purpose so they will not be covered here for now.
* *Merge Conds* (*Fusionner Conds* in French) ![BETA](https://img.shields.io/badge/BETA-FF1212) allows you to merge conds automatically. As of v1.4b the following modes are supported:
  * *AND* - requires both Conds to succeed for the merged Cond to
  * *OR* - requires either Cond to succeed
  * *XNOR/EQV* - suceeds if both Conds fail OR if both Conds suceed. That is, their state is equivalent.
  * *NAND* - succeeds unless both Conds do.
* *Compile Conds* (Compiler Conds in French) from v1.4b onwards allows you to compile completely custom CExpressions see the Compiler Syntax section for a guide on this. ![BETA](https://img.shields.io/badge/BETA-FF1212)
## Common CExpression Functions
There are **ALOT** of functions that CExpressions can call - over 118 in yw1 alone! This section will explain the most common ones you'll frequently see:

* `GetPhase()` - no params; returns the current Phase
  * This is calculated as `(chapter_no * 10,000) + sub_phase_no` (official names extracted from yw4 save files).
  * Take for instance `40010` this means the player is in Chapter 4, with a subphase of 10 which is near the start of the Chapter.
* `GameClear()` - no params; returns a Boolean value.
  * Returns 1 if the player has completed the main story (post-game isn't needed) else 0.
* `RunTrigger(hash: TriggerID)` - executes a Trigger by it's ID; always returns 1.
  * Scope-dependant as inside maps `RunTrigger` calls will prioritise the map triggers over common triggers.
* `IsHaveItem(hash: ItemID)` - returns a Boolean value.
  * Returns 1 if the player has the specified Item, else 0.
* `GetQuestPhase(hash: QuestID)` - returns the `QuestPhase` for the specified Quest.
  * `QuestPhase` is a measure of the player's progress within the Quest - it's mostly quest-specific but some exist for all Quests e.g. `00` means Quest not initiated and `FF`/`255` means the Quest in question has been completed.
* `GetGlobalByteFlag(hash: GlobalByteFlagID)` - gets the value of the `GlobalByteFlag` with the specified ID.
  * `GlobalByteFlag`s are defined in `FLAG_INFO_1` in `data/res/sys/flag_config_*.cfg.bin` and are a uint8 (unsigned 8-bit integer) meaning that they can be any integer from 0-255.
* `SetGlobalByteFlag(hash: GlobalByteFlagID, Value)` - sets the value of the `GlobalByteFlag` with the specified ID.
  * `GlobalByteFlag`s are defined in `FLAG_INFO_1` in `data/res/sys/flag_config_*.cfg.bin` and are a uint8 (unsigned 8-bit integer) meaning that they can be any integer from 0-255.
* `SetGlobalBitFlag(hash: GlobalBitFlagID, Value)` - sets the value of the `GlobalBitFlag` with the specified ID.
  * `GlobalBitFlag`s are defined in `FLAG_INFO_0` in `data/res/sys/flag_config_*.cfg.bin` and are Boolean meaning that they can be either `0` or `1`.
* `GetGlobalBitFlag(hash: GlobalBitFlagID)` - gets the value of the `GlobalBitFlag` with the specified ID.
  * `GlobalBitFlag`s are defined in `FLAG_INFO_0` in `data/res/sys/flag_config_*.cfg.bin` and are Boolean meaning that they can be either `0` or `1`.

## Compiler Syntax
The compiler accepts an expression formatted as `(3 + 1) > 5` - note that there is implicit operator precedence so `3 + 3 * 5` compiles to `3 + (3 * 5)` and not `(3 + 3) * 5`. The compiler accepts the following elements:
* Brackets `()` - these can be used to define explicit operator precedence
* Integer Literals `3` - (32-bit signed) these can be defined as normal.
* Float Literals `3f`, `3.0`, `Infinity`, `inf`, `NaN`, `-Infinity`, `-inf`, `NaNf`, `nanf`, `3e5` - these can be defined by using a decimal point, using the `f` keyword, or for special float values like `Infinity`, `NaN` and `-Infinity` they can be typed as is with the `f` keyword still being supported. Additionally the checks are *case-insensitive* so `InFINity` is just as valid as `Infinity` and its alias `inf`. Finally, [scientific notation](https://en.wikipedia.org/wiki/Scientific_notation) (also called exponential notation) is supported with lowercase and uppercase `e` i.e. `3e5` becomes `300000.0f` (3 * 10^5).
* Function calls `GetMoney()`, `FUNC_BF7BF3F5()` - these are identifiers followed by brackets without whitespace where parameters are expressions delimited by commas (`,`) - you can type their name i.e. `GetMoney()` and it'll automatically compute their CRC-32 hash - for functions where you do not know the original string - just like the decompiler output, `FUNC_` followed by the hash in the form of hexadecimal characters is a supported method i.e. `FUNC_BF7BF3F5()` uses the func with the CRC-32 hash `0xBF7BF3F5` aka `GetMoney()`.
* Operators - expects an [infix expression](http://en.wikipedia.org/wiki/Infix_notation) i.e. `3 + 2` not `3 2 + `. Additionally note that unary operators must be placed before the operand i.e. `++3`. As mentioned earlier, the compiler uses implicit operator precedence (e.g. multiplication has a higher precedence than addition so multiplication goes first; operators with the same precedence level execute from left-to-right unless otherwise noted). A list of operators with their precedence can be found below:
<div id="ops"></div> <!-- div for hyperlink to the non-technical op table -->

| Symbol | Op Count | Precedence | Category              | Operation          | Notes                                               |
| ------ | -------- | ---------- | --------------------- | ------------------ | --------------------------------------------------- |
| ++     | 1        | 14         | Unary Arithmetic      | Incrementation     | Unofficial symbol. Unary only.                      |
| --     | 1        | 14         | Unary Arithmetic      | Decrementation     | Unofficial symbol. Unary only.                      |
| ~      | 1        | 14         | Misc. Unary           | Bitwise NOT        | Unofficial symbol.                                  |
| (bool) | 1        | 14         | Misc. Unary           | To Bool            | Returns 1 if operand ≠ 0 else 0. Unofficial symbol. |
| *      | 2        | 13         | Binary Arithmetic     | Multiply           |                                                     |
| /      | 2        | 13         | Binary Arithmetic     | Divide             |                                                     |
| %      | 2        | 13         | Binary Arithmetic     | Modulus            |                                                     |
| +      | 2        | 12         | Binary Arithmetic     | Addition           |                                                     |
| -      | 2        | 12         | Binary Arithmetic     | Subtraction        |                                                     |
| <<     | 2        | 11         | Bit Shifting          | Left shift         |                                                     |
| >>     | 2        | 11         | Bit Shifting          | Right shift        |                                                     |
| <      | 2        | 10         | Relational Comparison | Less than          |                                                     |
| <=     | 2        | 10         | Relational Comparison | Less or equal      |                                                     |
| >      | 2        | 10         | Relational Comparison | Greater than       |                                                     |
| >=     | 2        | 10         | Relational Comparison | Greater or equal   |                                                     |
| ==     | 2        | 9          | Equality Comparison   | Equal              |                                                     |
| !=     | 2        | 9          | Equality Comparison   | Not equal          |                                                     |
| &      | 2        | 8          | Bitwise               | Bitwise AND        |                                                     |
| ^      | 2        | 7          | Bitwise               | Bitwise XOR        |                                                     |
| \|     | 2        | 6          | Bitwise               | Bitwise OR         |                                                     |
| &&     | 2        | 5          | Logical               | Logical AND        | By far the most common.                             |
| \|\|   | 2        | 4          | Logical               | Logical OR         | Frequently combined with `8F`.                      |
> Operator precedence matches C++ for consistency. Unary operators bind the tightest, while control-flow and logical operators have the lowest precedence and are evaluated last.

