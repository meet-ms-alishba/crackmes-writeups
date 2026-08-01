# Juche Loyalty Evaluation — Crackme Writeup

**Difficulty:** 1.2 | **Platform:** Linux x86-64 | **Tools:** GDB (GEF), Ghidra

## Challenge
Binary asks 2 questions; correct answers grant "party membership."

## Solution

1. **Static analysis (Ghidra):** Found `ptrace(PTRACE_TRACEME)` anti-debug
   check in `main`. In `take_loyalty_test`, found `strcmp(answer, correct_q1)`
   and `strcmp(answer, correct_q2)`. Decoded hardcoded answers (built
   char-by-char to dodge `strings`): `correct_q1 = "38"`, `correct_q2 = "Mount Paektu"`.

2. **Dynamic analysis (GDB):** Q1 ("38") passed, Q2 kept failing. Breakpointed
   the exact `strcmp` call (`0x4013ea`) and inspected registers:
   x/s $rdi → "Mount"
   x/s $rsi → "Mount Paektu"
3. **Root cause:** `cin >> answer` stops reading at whitespace — "Mount Paektu"
   only captured as "Mount". A classic C++ input gotcha, not a guessing error.

4. **Bypass:** Patched the buffer directly at the breakpoint:
   set {char[13]}$rdi = "Mount Paektu"
   continue
`strcmp` matched → party membership granted.

## Takeaway
`cin >>` truncates on whitespace; use `getline()` for multi-word input.
Runtime memory patching via GDB is a fast way to bypass a check once
the exact comparison point is located.
