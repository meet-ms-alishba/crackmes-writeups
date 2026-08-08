# SoulReaper — XorGate

**Difficulty:** ~1.5
**Tools used:** Ghidra

## Overview

Binary ("SoulReaper Crackme") asks for a username and a password, then
validates the password against a value derived from the username.

## Analysis

Decompiled logic showed the expected password is built as:

```
password = hex(username[i] XOR 0x23) for each character, concatenated
           + "@password"
```

`0x23` corresponds to `'#'` as the XOR key, and `"@password"` is a fixed
suffix appended at the end.

## Solution

Wrote a small script to reproduce the formula for any given username,
rather than calculating hex/XOR by hand (error-prone).

Example, username `Alishba`:

```
Password: 624f4a504b4142@password
```

Entering the username and calculated password:

```
[+] Access granted!
[+] FLAG{SoulReaper_XOR_Crackme}
```

## Reflections

Straightforward once the XOR key and suffix were identified from the
decompiled code — a good reminder that simple keyed-XOR password schemes
are common in beginner-to-intermediate crackmes, and scripting the formula
avoids manual calculation mistakes.
