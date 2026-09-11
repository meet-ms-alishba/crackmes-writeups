# TermBreaker Crackme -- Writeup

**Challenge:** TermBreaker by ShadowLegion  
**Platform:** crackmes.one  
**Difficulty:** 3.0  
**Language:** C++ / Qt6  
**Arch:** x86-64 Linux  

---

## Challenge Description

A retro terminal-themed crackme built with Qt6/C++.  
Goal: Find a valid access code **and** write a keygen that generates valid codes.  
A single hardcoded solution is not the intended solve.

---

## Tools Used

- `strings` -- initial recon
- Ghidra -- static analysis / decompilation
- Python -- keygen

---

## Step 1: Initial Recon

```bash
strings TermBreaker
```

Key findings:

```
OnSubmitClicked          ← button click handler
strcmp                   ← string comparison in use
ACCESS GRANTED!          ← success message
AUTHENTICATION FAILED - CODE REJECTED  ← fail message
AUTHENTICATING...        ← processing indicator
```

This told us:
- Validation triggers on button click (`OnSubmitClicked`)
- `strcmp` is imported -- some string comparison happens
- No packing, no anti-debug

---

## Step 2: Finding the Validation Function

In Ghidra:

```
Search → Memory → string: OnSubmitClicked
```

Found the string at `0x00108170`.  
Followed `XREF[1]` → landed on `FUN_001060b0`.

This is `MainWindow::OnSubmitClicked()` -- the actual validation function.

---

## Step 3: Reversing the Algorithm

Decompiled code (Ghidra):

```c
void FUN_001060b0(QWidget *param_1)
{
    ushort uVar1, uVar2, uVar3, uVar4, uVar5, uVar6, uVar7, uVar8;

    QLineEdit::text();          // read user input

    if (local_28 == 8) {        // must be exactly 8 chars

        uVar1 = local_30[0];
        // each char must be A-Z or 0-9
        if ((uVar1 - 0x41 < 0x1a) || ((ushort)(uVar1 - 0x30) < 10)) {
            uVar2 = local_30[1];
            // ... same check for all 8 chars ...

            // KEY CONDITION:
            if ((((uint)uVar5 * 5 +
                  (uint)uVar3 + (uint)uVar3 * 2 +
                  (uint)uVar2 * 2 + (uint)uVar1 +
                  (uint)uVar4 * 4 + (uint)uVar6 * 6 +
                  (uint)uVar7 * 8) - (uint)uVar7) +
                 (uint)uVar8 * 8 == 0xb28) {
                // ACCESS GRANTED
            }
        }
    }
}
```

**Simplified formula:**

```
c[0]*1 + c[1]*2 + c[2]*3 + c[3]*4 +
c[4]*5 + c[5]*6 + c[6]*7 + c[7]*8 == 0xB28 (2856)
```

**Constraints:**
- Length = exactly 8
- Each character: `A-Z` (0x41-0x5A) or `0-9` (0x30-0x39)

---

## Step 4: Writing the Keygen

Instead of brute-forcing all 8 characters, we fix the first 7 and
mathematically derive the 8th:

```python
#!/usr/bin/env python3
"""
TermBreaker Keygen
Algorithm: c[0]*1 + c[1]*2 + ... + c[7]*8 == 0xB28
"""

TARGET = 0xB28  # 2856

def is_valid_char(c):
    return 0x41 <= c <= 0x5A or 0x30 <= c <= 0x39

def generate_codes(count=10):
    found = []
    for c0 in range(0x41, 0x5B):
        for c1 in range(0x41, 0x5B):
            for c2 in range(0x41, 0x5B):
                for c3 in range(0x41, 0x5B):
                    for c4 in range(0x41, 0x5B):
                        for c5 in range(0x41, 0x5B):
                            for c6 in range(0x41, 0x5B):
                                partial = (c0*1 + c1*2 + c2*3 + c3*4 +
                                           c4*5 + c5*6 + c6*7)
                                rem = TARGET - partial
                                if rem > 0 and rem % 8 == 0:
                                    c7 = rem // 8
                                    if is_valid_char(c7):
                                        code = ''.join(chr(c) for c in
                                               [c0,c1,c2,c3,c4,c5,c6,c7])
                                        found.append(code)
                                        if len(found) >= count:
                                            return found
    return found

if __name__ == "__main__":
    codes = generate_codes(10)
    print("Valid codes:")
    for code in codes:
        print(f"  {code}")
```

---

## Step 5: Results

```
Valid codes:
  AAAABYZV
  AAAABZXZ
  AAAACXYZ
  AAAADVZZ
  AAAADZZW
```

All of these pass the validation algorithm.

---

## Key Takeaways

- Binary never needed to run -- pure static analysis was sufficient
- `strings` quickly revealed the entry point (`OnSubmitClicked`)
- Ghidra decompiler made the checksum formula readable
- The algorithm is a simple weighted sum -- easily reversible
- Infinite valid codes exist (any combination satisfying the equation)

---

## Notes

Binary requires Qt 6.11 which was not available on the test system (Kali, Qt 6.3).  
Static analysis alone was sufficient to fully reverse and solve the challenge.
