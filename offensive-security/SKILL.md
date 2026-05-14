---
name: offensive-security
description: Educational exploitation knowledge for authorized security testing, CTF challenges, and security research. Covers buffer overflows, format strings, shellcode, network attacks, cryptology, and modern countermeasures.
origin: ECC
---

# Offensive Security

> **NOTICE**: This skill is strictly for educational purposes, authorized penetration testing, CTF competitions, and defensive security research. Always obtain proper written authorization before testing any system you do not own. Unauthorized access to computer systems is illegal.

## Overview

This skill provides deep exploitation knowledge drawn from foundational security research. It enables Claude Code to assist with:

- Understanding how vulnerabilities work at the memory and protocol level
- Solving CTF (Capture The Flag) challenges involving binary exploitation, reverse engineering, and network analysis
- Reviewing code for security vulnerabilities with exploitation-aware insight
- Writing secure code by understanding how attackers think
- Explaining modern exploit mitigations and their bypass techniques
- Analyzing real-world CVEs and vulnerability disclosures

## When to Activate

Activate this skill when the user is working on:

- **CTF challenges** — binary exploitation, pwn challenges, reverse engineering, crypto challenges, network forensics
- **Security code review** — identifying buffer overflows, format string bugs, use-after-free, integer overflows, race conditions
- **Understanding vulnerabilities** — explaining how a specific CVE or exploit technique works at the technical level
- **Writing secure code** — applying defensive patterns informed by knowledge of attack techniques
- **Penetration testing concepts** — understanding exploitation methodology for authorized engagements
- **Analyzing exploit mitigations** — evaluating whether ASLR, NX, canaries, RELRO, CFI are properly deployed and how they can fail

---

## Topics Index

### Programming for Exploitation

**C Essentials for Security Research**

- **Pointers**: Direct memory access, pointer arithmetic, function pointers, void pointers. Pointers are the foundation of nearly every memory corruption exploit.
- **Arrays and Strings**: C arrays have no bounds checking. Strings are null-terminated char arrays. The gap between allocated size and actual bounds checking is where buffer overflows live.
- **Memory Allocation**: `malloc`/`free`/`realloc` and the heap allocator internals (chunk headers, free lists, bins). Understanding allocator metadata is essential for heap exploitation.
- **Type Casting and Integer Issues**: Implicit conversions, sign extension, integer overflow/underflow. These often gate whether a bounds check can be bypassed.

**x86 Assembly Fundamentals**

- **Registers**: General-purpose (EAX, EBX, ECX, EDX), index (ESI, EDI), stack (ESP, EBP), instruction pointer (EIP). In x86-64: RAX, RBX, ..., RSP, RBP, RIP, plus R8-R15.
- **Calling Conventions**: cdecl (caller cleans stack, args right-to-left on stack), System V AMD64 ABI (args in RDI, RSI, RDX, RCX, R8, R9, then stack). Return values in EAX/RAX.
- **Stack Frames**: Function prologue (`push ebp; mov ebp, esp; sub esp, N`), epilogue (`leave; ret`). Local variables at negative offsets from EBP, arguments at positive offsets.
- **Common Instructions**: `mov`, `push`, `pop`, `call`, `ret`, `lea`, `cmp`, `jmp`, `jne/je/jg/jl`, `xor`, `int 0x80` (32-bit syscall), `syscall` (64-bit).

**Memory Layout**

```
High Address
+------------------+
|    Stack         |  Grows downward (toward lower addresses)
|    (LIFO)        |  Local variables, return addresses, saved registers
+------------------+
|       |          |
|       v          |
|                  |
|       ^          |
|       |          |
+------------------+
|    Heap          |  Grows upward (toward higher addresses)
|    (dynamic)     |  malloc/free managed memory
+------------------+
|    BSS           |  Uninitialized global/static variables (zeroed)
+------------------+
|    Data          |  Initialized global/static variables
+------------------+
|    Text          |  Executable code (read-only)
+------------------+
Low Address
```

**Debugging with GDB**

- **Breakpoints**: `break main`, `break *0x08048456`, `break function_name`
- **Execution**: `run`, `continue`, `stepi` (single instruction), `nexti` (step over calls), `finish` (run until return)
- **Memory Examination**: `x/20x $esp` (20 hex words from stack pointer), `x/s 0x08048500` (string at address), `x/10i $eip` (10 instructions from instruction pointer)
- **Disassembly**: `disas main`, `set disassembly-flavor intel`
- **Info Commands**: `info registers`, `info frame`, `info proc mappings`, `info functions`
- **GDB Extensions**: PEDA, GEF, pwndbg add exploit-dev features (pattern create/offset, checksec, heap analysis, ROP gadget search)

---

### Buffer Overflow Exploitation

**Stack-Based Buffer Overflows**

The classic vulnerability: writing past the end of a stack buffer overwrites the saved return address. When the function returns, execution jumps to the attacker-controlled address.

- **Return Address Overwrite**: Overflow a local buffer to overwrite EIP/RIP saved on the stack. Redirect execution to injected shellcode or a known function.
- **NOP Sleds**: Pad shellcode with NOP instructions (`0x90`) to create a large landing zone. The return address only needs to land somewhere in the sled to slide into the payload.
- **Finding the Offset**: Use cyclic patterns (e.g., `pattern_create` / `pattern_offset` in GEF/pwndbg, or `msf-pattern_create`) to determine exact offset to saved return address.

**Heap-Based Overflows**

- **Metadata Corruption**: Overflowing a heap chunk can corrupt the size/flags or forward/backward pointers of adjacent chunks. When `free()` processes corrupted metadata, it can perform an arbitrary write (unlink attack).
- **Use-After-Free**: Accessing memory after it has been freed. If the freed memory is reallocated for a different object, the dangling pointer now references attacker-controlled data.
- **Double Free**: Freeing the same chunk twice corrupts the allocator's free list, potentially allowing allocation of overlapping chunks.

**Off-By-One Exploitation**

- **Frame Pointer Overwrite**: An off-by-one error in a stack buffer can overwrite the least significant byte of the saved EBP. When the calling function uses this corrupted EBP to restore ESP and then returns, EIP is loaded from attacker-influenced memory.

**Return-to-libc**

- **Concept**: Instead of injecting code, reuse existing functions in libc (like `system()`, `execve()`). The stack is set up so that when the function returns, it "calls" `system("/bin/sh")`.
- **Bypasses NX/DEP**: No code injection needed since the executed code already exists in legitimate executable memory.
- **Stack Setup**: Overwrite return address with address of `system()`, followed by a fake return address (or address of `exit()`), followed by pointer to `"/bin/sh"` string.

See: [references/buffer-overflow-techniques.md](references/buffer-overflow-techniques.md)

---

### Format String Vulnerabilities

**How printf Works**

`printf(user_input)` is the vulnerability. When the format string is user-controlled, the attacker controls how `printf` interprets the stack.

**Format Specifiers for Exploitation**

- `%x` — Read 4 bytes from the stack (treats next stack value as unsigned hex integer)
- `%s` — Read a string from the address pointed to by the next stack value
- `%n` — **Write** the number of characters printed so far to the address pointed to by the next stack value
- `%hn` — Write a short (2 bytes) instead of 4
- `%hhn` — Write a single byte
- `%N$x` — Direct parameter access: read the Nth argument from the stack
- `%N$n` — Direct parameter access: write to the address at the Nth position

**Exploitation Technique**

1. **Stack Reading**: Use `%x` repeatedly (or `%N$x` for direct access) to leak stack contents, including return addresses (defeating ASLR) and canary values.
2. **Arbitrary Write with %n**: Control the number of characters printed (via padding like `%100x`), then use `%n` to write that count to an address you place on the stack.
3. **Short Writes**: Use `%hn` to write 2 bytes at a time. This avoids printing billions of characters for large values. Write the target value in two halves to two consecutive addresses.
4. **GOT Overwrite**: Overwrite a Global Offset Table entry so the next call to that library function jumps to your shellcode or to `system()`.

---

### Shellcode Development

**System Call Interface**

- **Linux x86 (32-bit)**: Syscall number in EAX, args in EBX, ECX, EDX, ESI, EDI. Trigger with `int 0x80`.
- **Linux x86-64**: Syscall number in RAX, args in RDI, RSI, RDX, R10, R8, R9. Trigger with `syscall`.
- **Key Syscalls**: `execve` (11/59), `read` (3/0), `write` (4/1), `open` (5/2), `socket` (359/41), `connect` (362/42), `dup2` (63), `exit` (1/60).

**Common Shellcode Patterns**

- **/bin/sh Spawn**: `execve("/bin/sh", NULL, NULL)` — the minimal useful payload
- **Port-Binding Shell**: `socket()` -> `bind()` -> `listen()` -> `accept()` -> `dup2()` for stdin/stdout/stderr -> `execve("/bin/sh")`
- **Reverse Shell**: `socket()` -> `connect()` to attacker IP/port -> `dup2()` -> `execve("/bin/sh")`

**Null Byte Avoidance**

Shellcode injected via string functions (strcpy, gets) cannot contain null bytes (`0x00`) since they terminate the string.

- Replace `mov eax, 0` with `xor eax, eax`
- Replace `mov al, 11` (if upper bytes might be non-zero) with `xor eax, eax; mov al, 11`
- Use `push` and stack construction instead of hardcoded addresses with nulls
- Replace `/bin/sh\0` push with calculated XOR values

**Polymorphic Shellcode**

- **XOR Encoding**: XOR the payload with a key, prepend a decoder stub. At runtime the stub decodes the payload in-place, then executes it.
- **Purpose**: Evade signature-based IDS/IPS that look for known shellcode byte patterns.

See: [references/shellcode-reference.md](references/shellcode-reference.md)

---

### Network Attacks

**Socket Programming Fundamentals**

- **TCP Client**: `socket()` -> `connect()` -> `send()`/`recv()` -> `close()`
- **TCP Server**: `socket()` -> `bind()` -> `listen()` -> `accept()` -> `send()`/`recv()` -> `close()`
- **UDP**: Connectionless. `socket()` -> `sendto()`/`recvfrom()`
- **Raw Sockets**: `socket(AF_INET, SOCK_RAW, protocol)` for crafting custom packets at the IP level. Requires root/CAP_NET_RAW.

**Protocol Analysis**

- **IP Header**: Version, IHL, total length, identification, flags (DF, MF), fragment offset, TTL, protocol, checksum, source/destination addresses (20 bytes minimum).
- **TCP Header**: Source/destination ports, sequence number, acknowledgment number, data offset, flags (SYN, ACK, FIN, RST, PSH, URG), window size, checksum (20 bytes minimum).
- **TCP Three-Way Handshake**: SYN -> SYN-ACK -> ACK. Sequence numbers are exchanged and incremented.
- **UDP Header**: Source/destination ports, length, checksum (8 bytes).

**Network Sniffing**

- **Promiscuous Mode**: Network interface captures all traffic on the segment, not just traffic addressed to it.
- **libpcap/pcap**: Standard library for packet capture. `pcap_open_live()`, `pcap_loop()`, `pcap_compile()` for BPF filters.
- **BPF Filters**: `tcp port 80`, `host 192.168.1.1`, `tcp[13] & 2 != 0` (SYN flag set).

**Denial of Service**

- **SYN Flood**: Send massive numbers of SYN packets with spoofed source addresses. The target allocates resources for half-open connections, exhausting its connection table.
- **Amplification Attacks**: Send small requests with spoofed source IP to services that reply with much larger responses (DNS, NTP, memcached). The amplified traffic overwhelms the victim.

**Man-in-the-Middle**

- **ARP Spoofing**: Send gratuitous ARP replies to associate the attacker's MAC address with the gateway's IP. Traffic intended for the gateway flows through the attacker.
- **DNS Poisoning**: Inject forged DNS responses to redirect domain lookups to attacker-controlled servers.

See: [references/network-attack-patterns.md](references/network-attack-patterns.md)

---

### Countermeasures

| Mitigation | How It Works | Bypass Technique |
|---|---|---|
| **Stack Canaries** | Random value placed between local variables and saved return address. Checked before function returns. | Information leak (format string, read primitive) to discover canary value, then include it in overflow payload. |
| **ASLR** | Randomizes base addresses of stack, heap, libraries, and (with PIE) the executable on each run. | Information leak to discover a runtime address, then calculate offsets. Partial overwrites (only overwrite lower bytes that aren't randomized). Brute force on 32-bit (limited entropy). |
| **NX / DEP** | Marks stack and heap as non-executable. Injected shellcode cannot run. | Return-Oriented Programming (ROP): chain snippets of existing executable code ("gadgets") ending in `ret` to perform arbitrary computation. Return-to-libc as a simpler variant. |
| **RELRO** | Partial: GOT is writable but relocations happen at load time for some entries. Full: entire GOT is read-only after startup. | Partial RELRO: GOT overwrite still works for lazily-bound entries. Full RELRO: must find alternative write targets (.fini_array, __malloc_hook, stack). |
| **Fortify Source** | Compile-time and runtime checks for buffer overflow in common functions (memcpy, strcpy, printf). Replaces unsafe calls with checked versions. | Only covers known dangerous functions. Custom code, non-standard functions, and complex control flow may not be fortified. |
| **PIE** | Position-Independent Executable. Allows ASLR to randomize the executable's base address (not just libraries). | Same as ASLR bypass: information leak to discover base address. Without a leak, partial overwrites on lower 12 bits (page offset) are still deterministic. |
| **CFI** | Control-Flow Integrity. Restricts indirect calls/jumps to a set of valid targets determined at compile time. | Targets within the valid set may still be useful for exploitation (COOP — Counterfeit Object-Oriented Programming). Implementation gaps in coverage. |

**ROP (Return-Oriented Programming) in Detail**

ROP chains existing code "gadgets" — short instruction sequences ending in `ret` — to build arbitrary computation without injecting code.

1. Find gadgets with tools like `ROPgadget`, `ropper`, or `xgadget`
2. Chain gadgets on the stack: each `ret` pops the next gadget address
3. Common patterns: `pop rdi; ret` (set first argument), `pop rsi; pop r15; ret` (set second argument), then call `system()` or `execve()`

See: [references/countermeasures-bypasses.md](references/countermeasures-bypasses.md)

---

### Cryptology

**Symmetric Encryption**

- **XOR Cipher**: Simplest cipher. `ciphertext = plaintext XOR key`. Vulnerable to known-plaintext attacks, frequency analysis if key is short.
- **DES**: 64-bit blocks, 56-bit key (too short). Feistel network, 16 rounds. Triple-DES (3DES) applies DES three times with 2-3 keys.
- **AES**: 128-bit blocks, 128/192/256-bit keys. Substitution-permutation network, 10/12/14 rounds. Current standard.
- **Block Cipher Modes**: ECB (no chaining, reveals patterns), CBC (XOR with previous ciphertext block, IV required), CTR (turns block cipher into stream cipher), GCM (authenticated encryption).

**Asymmetric Encryption**

- **RSA**: Based on difficulty of factoring large semiprimes. Key generation: choose primes p, q; n = p*q; e (public exponent, commonly 65537); d = e^(-1) mod phi(n). Encrypt: c = m^e mod n. Decrypt: m = c^d mod n.
- **Diffie-Hellman**: Key exchange protocol. Both parties agree on prime p and generator g. Each picks a secret exponent, computes public value, exchanges. Shared secret = g^(ab) mod p. Vulnerable to MITM without authentication.

**Hashing**

- **MD5**: 128-bit hash. Broken — collision attacks are practical (seconds on modern hardware).
- **SHA-1**: 160-bit hash. Broken — collision demonstrated (SHAttered, 2017).
- **SHA-256/SHA-3**: Current standards. No known practical attacks.
- **Collision Attacks**: Finding two inputs that produce the same hash. Birthday paradox means collisions are found in O(2^(n/2)) rather than O(2^n).
- **Length Extension**: MD5, SHA-1, SHA-256 are vulnerable to length extension attacks (Merkle-Damgard construction). SHA-3 and HMAC are not.

**Wireless Security**

- **WEP**: Broken. Uses RC4 with a 24-bit IV. IV reuse and related-key attacks allow key recovery from captured traffic (aircrack-ng).
- **WPA/WPA2-PSK**: Uses PBKDF2 to derive key from passphrase. Vulnerable to offline dictionary attacks against captured 4-way handshake. WPA-TKIP has additional vulnerabilities.
- **WPA2-Enterprise**: Uses 802.1X/RADIUS for authentication. Significantly more secure than PSK.
- **WPA3**: SAE (Simultaneous Authentication of Equals) replaces PSK exchange. Resistant to offline dictionary attacks. Dragonblood vulnerabilities found in early implementations.

---

## Key Concepts

### Memory Layout Diagram (Process Exploitation View)

```
+========================+  0xFFFFFFFF (high memory)
|      Kernel Space      |  (inaccessible to user programs)
+========================+  0xC0000000 (typical 32-bit split)
|        Stack           |  <- ESP/RSP (grows DOWN)
|  [return addr]         |  <- Target for stack overflow
|  [saved EBP ]          |  <- Target for off-by-one
|  [local vars ]         |
|  [canary     ]         |  <- Stack protector
|        ...             |
+------------------------+
|   Memory-Mapped Region |  Shared libraries (libc, ld-linux)
|   (mmap)               |  <- ASLR randomizes base
+------------------------+
|        Heap            |  <- Grows UP
|  [chunk header]        |  <- Target for heap overflow
|  [user data   ]        |
|  [chunk header]        |
+------------------------+
|        BSS             |  Uninitialized globals (zeroed)
+------------------------+
|        Data            |  Initialized globals
+------------------------+
|        Text            |  Program code (read-only, executable)
+========================+  0x08048000 (typical 32-bit ELF base)
```

### Exploit Development Workflow

1. **Reconnaissance**: Identify the target binary, OS, architecture, protections (`checksec`)
2. **Vulnerability Discovery**: Fuzzing, source review, reverse engineering (IDA, Ghidra, Binary Ninja)
3. **Crash Analysis**: Reproduce the crash, identify the root cause (buffer overflow, format string, use-after-free)
4. **Control EIP/RIP**: Determine exact offset to overwrite the instruction pointer or other control data
5. **Bypass Mitigations**: Defeat ASLR (leak), NX (ROP), canaries (leak or brute force), RELRO (alternative targets)
6. **Payload Development**: Write or generate shellcode/ROP chain for the desired action
7. **Reliability**: Handle ASLR variance, ensure payload works across runs, add error recovery
8. **Testing**: Verify in controlled environment before any authorized engagement

---

## Problem-Solving Patterns

| Problem | Pattern | Tools / Techniques |
|---|---|---|
| Binary crashes but no control of EIP | Find exact overflow offset | `pattern_create`/`pattern_offset`, cyclic patterns, GDB/GEF |
| NX enabled, can't execute stack shellcode | Return-Oriented Programming | `ROPgadget`, `ropper`, `pwntools` ROP module |
| ASLR enabled, addresses change each run | Information leak | Format string `%p` leak, partial overwrite, `puts@plt` to leak GOT |
| Stack canary blocking overflow | Leak or bypass canary | Format string leak, brute-force (forking servers), overwrite non-canary-protected path |
| Full RELRO, can't overwrite GOT | Alternative write targets | `.fini_array`, `__malloc_hook`, `__free_hook`, stack return address |
| Shellcode has null bytes | Encode or rewrite | XOR encoder, `xor eax, eax` instead of `mov eax, 0`, push/pop tricks |
| Remote exploit, no output channel | Blind exploitation | Timing side-channels, out-of-band data exfil (DNS), brute force byte-by-byte |
| Heap exploitation needed | Understand allocator | `malloc`/`free` internals, tcache poisoning, fastbin dup, house-of-* techniques |
| CTF challenge with unknown vulnerability | Systematic analysis | `checksec`, `file`, `strings`, `ltrace`/`strace`, Ghidra decompilation |

## Common Pitfalls

| Pitfall | Why It Fails | Fix |
|---|---|---|
| Hardcoded addresses in exploit | ASLR changes addresses each run | Leak addresses at runtime, calculate offsets from leaked base |
| Shellcode contains null bytes | `strcpy`/`gets` stops copying at `\x00` | Use null-free encoding (XOR, register tricks, push/pop) |
| Wrong endianness in address packing | x86 is little-endian | Use `struct.pack('<I', addr)` or `p32()`/`p64()` from pwntools |
| Forgetting stack alignment | x86-64 requires 16-byte RSP alignment before `call` | Add an extra `ret` gadget before function call to realign |
| Testing on wrong libc version | Offsets differ between libc versions | Match target libc exactly, use `libc-database` or `patchelf` |
| Ignoring PIE in offset calculations | Executable base is randomized with PIE | Leak executable base address, add base to all text/plt/got offsets |
| ROP chain clobbers needed registers | Gadgets may have side effects | Carefully trace register state through each gadget, pick clean gadgets |
| Format string offset miscalculated | Stack layout varies by compiler/optimization | Empirically determine offset with `AAAA%p%p%p...` and find the `0x41414141` |

---

## Source Material

- **Hacking: The Art of Exploitation, 2nd Edition** by Jon Erickson (No Starch Press, 2008)
  - Comprehensive coverage of exploitation from first principles
  - Includes LiveCD environment for hands-on practice
  - Covers C programming, x86 assembly, buffer overflows, format strings, shellcode, networking, and cryptology

> This skill synthesizes concepts from the source material for educational and authorized defensive security use. Always ensure you have proper authorization before applying any offensive techniques.
