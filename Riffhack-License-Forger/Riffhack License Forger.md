
This was a reverse engineering challenge involving a license key validator written in C, compiled to a stripped x86-64 ELF binary.

## Recon

First, basic recon on the file:

```bash
file riffhack_license_forger
```

```
ELF 64-bit LSB pie executable, x86-64, ... stripped
```

Running it blind shows it's an interactive license key checker:

```
riffhack license forger // paid builder unlock
vendor: riffhack-labs
enter activation key to unlock the builder
license>
```

Since there's no debug symbols, I pulled strings first to get a feel for the program flow:

```bash
strings -n 6 riffhack_license_forger
```

This immediately leaked a decoy key in plaintext:

```
RH-DEMO-
paid builder receipt unlocked:
demo license accepted. decrypted trial receipt:
stage one accepted. builder secret recovered:
builder secret>
RH-DEMO-TRIAL
```

## The decoy

Trying `RH-DEMO-TRIAL` confirmed it's a trap — it unlocks a fake "demo" path:

```
demo license accepted. decrypted trial receipt:
demo receipt only - no payout
```

So the real key had to satisfy a separate validator elsewhere in the binary.

## Disassembly

I disassembled with `objdump -d -M intel` and manually traced the key-checking function, since there was no decompiler available in the environment. The validator works on a 20-byte key and checks (among other things):

- `key[2] == key[8] == key[13] == '-'` (dash-separated format)
- Byte-level arithmetic constraints between several positions (XORs and sums between specific indices)
- A literal substring `"OPEN"` at a fixed offset
- A 4-digit decimal field that, when parsed as an integer, must equal `2026`
- A final FNV-style rolling hash over the whole 20 bytes that must equal a hardcoded constant `0xc7662a57`

## Solving the constraints

I wrote a small Python script to encode each of these constraints and solve for the unknown bytes directly (most of them were fully determined just by chaining the relations, no brute force needed), then verified the hash matched:

```python
def keyhash(s):
    h = 0x8f31a5
    for i in range(20):
        b = s[i]
        h ^= (b + i * 0x45) & 0xff
        h = (h * 0x1000193) & 0xffffffff
        h = ((h << 5) | (h >> 27)) & 0xffffffff
    return h
```

This resolved to a unique valid key:

```
RH-TRIAL-2026-OPEN!!
```

## Stage one

Feeding that to the binary unlocked stage one and decrypted an embedded "builder secret" (XOR'd against a keystream derived from the key bytes):

```bash
echo "RH-TRIAL-2026-OPEN!!" | ./riffhack_license_forger
```

```
stage one accepted. builder secret recovered:
FORGE-STAGE-2
```

## Stage two

The program then prompts for that exact secret a second time, which triggers decryption of a second embedded blob — the real receipt:

```bash
printf "RH-TRIAL-2026-OPEN!!\nFORGE-STAGE-2\n" | ./riffhack_license_forger
```

```
paid builder receipt unlocked:
bitctf{r1ff_l1c3n53_m4k3r_unl0ck3d}
```

## Flag

```
bitctf{{r1ff_l1c3n53_m4k3r_unl0ck3d}}
```
