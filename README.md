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

# Roadmap
## v1.5
* This update will mainly focus on making things easier to use; the lib will be easily separable from the UI, there will be more configs etc
  * This will be done in preparation for being built into `yw-mod`, an upcoming project I'm working on!
