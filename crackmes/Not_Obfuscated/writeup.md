Challenge Overview The target binary not_obfuscated is a stripped 64-bit Linux ELF binary. Despite its name, it employs a combination of dynamic runtime unpacking via mprotect and an internal execution engine—specifically, a custom stack-based Clac VM interpreter running a pre-trained MNIST Neural Network for input validation.Phase 1: Triage & ReconnaissanceInitial analysis using standard Linux tools revealed protection mechanisms and indications of dynamic payload unpacking:Bash$ file crackme
crackme: ELF 64-bit LSB pie stripped executable, x86-64, version 1 (SYSV), dynamically linked

$ checksec --file=crackme
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
    Symbols:  No Symbols

$ strings crackme | grep -E "mmap|mprotect|password"
mmap failed
mprotect failed
password:
nope
correct!
Key Findings:mmap / mprotect calls suggest self-modifying code or runtime decryption.Plaintext strings (password:, nope, correct!) confirm the validation targets.Phase 2: Static Analysis (Ghidra)Since symbols are stripped, tracing started from the entry function to identify the actual main routine passed to __libc_start_main:C// Entry point resolving real main
__libc_start_main(FUN_00101100, argc, argv, ...);
Unpacking Routine in FUN_00101100 (main)Decompiling FUN_00101100 showed the runtime allocation and execution workflow:C// Allocation of executable memory region
void *mem = mmap(NULL, 0x1cb92, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

// Copying encrypted payload into memory
memcpy(mem, &encrypted_payload_data, 0x1cb92);

// Granting execution permission (PROT_READ | PROT_EXEC = 0x5)
mprotect(mem, 0x1cb92, PROT_READ | PROT_EXEC);

// Transferring control flow to the decrypted buffer
(**(code **)(auStack_40058._0_8_ + 0x610))(aiStack_40038);
Phase 3: Dynamic Unpacking & Memory Extraction (GDB)Because PIE was enabled, base addresses were resolved dynamically at runtime.Step 1: Base Offset ResolutionBashgdb ./crackme
(gdb) start
(gdb) info proc mappings
# Base Address: 0x555555554000
Step 2: Intercepting Execution at mprotectThe mprotect call was located at relative offset 0x1342. Calculating the target execution breakpoint:$$\text{Runtime Breakpoint} = 0x555555554000 + 0x1347 = 0x555555555347$$Bash(gdb) break *0x555555555347
(gdb) continue

# Inspecting argument registers passed to mprotect (RDI = addr, RSI = len, RDX = prot)
(gdb) info registers rdi rsi rdx
# RDI = 0x7ffff7f9f000 (Decrypted Memory Base)
# RSI = 0x1cb92        (Length)
# RDX = 0x5            (PROT_READ | PROT_EXEC)
Step 3: Dumping Decrypted Buffer to DiskBash(gdb) dump binary memory decrypted.bin 0x7ffff7f9f000 0x7ffff7fbca92
Phase 4: Interpreter Identification & Validation LogicInspecting the extracted decrypted.bin binary reveals internal Clac VM error handler strings:Plaintextbad opcode
undefined function
if overrun
missing function
The Clac VM ArchitectureThe decrypted payload implements an interpreter for Clac—a stack-based language. Embedded inside the Clac bytecode are pre-trained weights and biases forming a Neural Network (MNIST Classifier).Input SpecificationFormat: 49 integer values ($0\text{--}10$) passed line-by-line.Semantics: Represents a $7 \times 7$ downsampled pixel grid of a handwritten digit.Validation Criteria: Matrix multiplication ($W \cdot X + b$) must produce a confidence score classifying the input grid as Digit 1.Phase 5: Solution & Input VerificationTo satisfy the network's classification threshold for Digit 1, the following 49-element pixel vector was supplied:Plaintext1 8 10 5 10 8 9 8 0 5 0 6 6 7 2 7 0 7 5 1
0 1 1 1 10 6 9 4 5 1 9 4 4 8 10 8 4 1 5 3
8 7 5 4 2 1 7 9 4
Execution ResultBash$ ./crackme < input.txt
password:
correct!
Summary & Key TakeawaysUnpacking Strategy: When encountering mmap + memcpy + mprotect patterns, place breakpoints right before memory permission transitions (PROT_EXEC) to dump clean payload dumps.Interpreter Obfuscation: The presence of custom stack loops and opcode error strings signals a VM-based interpreter wrapper.Static vs Dynamic Logic: Neural networks inside binaries function as deterministic math formulas ($W \cdot X + b$). Reversing them relies on extracting the input schema rather than analyzing raw neural weights.
