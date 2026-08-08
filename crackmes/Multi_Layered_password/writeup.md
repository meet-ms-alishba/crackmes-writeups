# Multi-layer Password Check — by Mazzotti

**Difficulty:** 1.8
**Category:** Reverse Engineering / C++ Logic Analysis
**Tools used:** Ghidra, objdump, GDB (blocked by anti-debug), custom Ghidra-to-LLM decompiler cleanup tool (My tool Link in github)

## Challenge Overview

The binary asks the user to input 6 strings (no spaces) and validates them against
some internal logic. Getting all 6 correct prints a success message; getting any
wrong prints a failure message and exits.

## Static Analysis

Running the binary reveals the entry point in `main()`:

```
INFO: You have to enter 6 correct strings (without any spaces) to win.
GOOD LUCK!
Input string number 0 here >>>
```

Six strings are read via `std::cin >> input`, stored in a `std::string[6]` array,
then passed to a validation routine. If validation fails, the program prints
`Skill issue. Try harder mate!` and exits with status 1. On success, it prints
`Good job. Hmmmm. :3` and exits with status 0.

## Tool-Assisted Cleanup

Manually reading Ghidra's raw decompiled output for the validation function
would normally take a while, since it's full of generic variable names
(`puVar4`, `piVar3`, `local_70`) and low-level pointer arithmetic. I ran the
binary through a small custom tool I built (Ghidra headless export → LLM
cleanup pass) and it gave me a clean, readable version of the decompiled C
code almost immediately — turning what would've taken a while into a couple of
minutes.

The cleaned output revealed the validation function checks a list of 6
string entries, where each string must:
1. Have a length that's a multiple of 4
2. Consist entirely of repeated `"MAZZ"` blocks (no extra characters or spaces)
3. Have a block count that matches an expected value from a separate array

**Note:** my tool just reads the Ghidra decompiled text and makes its own
guesses about struct sizes/offsets from that — it doesn't have access to
actual runtime memory. In this case it guessed a 16-byte struct stride when
the real one (confirmed via raw assembly) was 32 bytes. So it's a good
practice to always verify offsets/sizes against the raw Ghidra output before
relying on tool.

## Verifying the Real Layout

Cross-checking the raw decompiled assembly:

```c
puVar4 = puVar4 + 4;   // pointer advances by 4 words = 32 bytes per entry
```

```
0x104340 - 0x104280 = 0xC0 = 192 bytes
192 / 32 bytes per entry = 6 entries
```

This confirmed 6 entries, matching the "enter 6 strings" prompt from `main()`.

## Getting the Expected Block Counts

GDB flagged an anti-debug check when trying to set a breakpoint, so I pulled
the expected block-count array statically instead:

```
objdump -s -j .data ./crackme
```

```
4010: 03000000 07000000 0c000000 01000000
4020: 0f000000 07000000
```

Decoded (little-endian): `[3, 7, 12, 1, 15, 7]`

## Solution

Each number is how many times `"MAZZ"` needs to repeat for that entry:

| Entry | Blocks | String |
|---|---|---|
| 0 | 3  | `MAZZMAZZMAZZ` |
| 1 | 7  | `MAZZMAZZMAZZMAZZMAZZMAZZMAZZ` |
| 2 | 12 | `MAZZ` × 12 |
| 3 | 1  | `MAZZ` |
| 4 | 15 | `MAZZ` × 15 |
| 5 | 7  | `MAZZMAZZMAZZMAZZMAZZMAZZMAZZ` |

Entering these in order produced:

```
Good job. Hmmmm. :3
```

## Reflections

This one was less about exploitation and more about efficiently reading
decompiled logic and correlating it with static data in the binary. The
GDB anti-debug check was a good reminder that static analysis (`objdump`)
is always a reliable fallback when dynamic analysis is blocked.
