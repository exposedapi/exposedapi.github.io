# PicoCTF 2024

## Overview

A multi-week online CTF hosted by Carnegie Mellon. Competed solo, focused on web exploitation and reverse engineering tracks.

---

## Web — `elements` (250 pts)

Classic client-side challenge. The flag was hidden across multiple HTTP responses — you had to collect headers from a sequence of redirects and assemble them in order.

```bash
curl -v --location-trusted https://challenge.picoctf.net/elements
```

The trick was that `--location-trusted` follows redirects while preserving the auth header. Each hop added one segment to the flag.

![Response chain showing the X-Flag-Part headers across three redirects](./screenshots/picoctf-2024-elements-redirects.png)

---

## Reverse Engineering — `weirdsnake` (300 pts)

A Python bytecode challenge. The `.pyc` was decompiled using `pycdc` — the result was obfuscated with custom string XOR.

```python
# decompiled snippet
key = [0x41, 0x6e, 0x61, 0x63, 0x6f, 0x6e, 0x64, 0x61]
enc = [0x13, 0x0e, 0x12, 0x01, 0x1f, 0x08, 0x1d, 0x01]
flag = ''.join(chr(a ^ b) for a, b in zip(key, enc))
```

Running it gives the first chunk. The second half was embedded in a nested lambda — took a while to untangle.

![Ghidra showing the decompiled lambda chain](./screenshots/picoctf-2024-weirdsnake-ghidra.png)

> the moment you realise the lambda is recursive — that's when it clicks.

---

## Crypto — `rsa_oracle` (400 pts)

Padding oracle on a textbook RSA implementation. The server would decrypt any ciphertext and leak whether the plaintext started with `\x00\x02` — classic PKCS#1 v1.5 oracle.

1. Capture the encrypted flag ciphertext
2. Use Bleichenbacher's attack to iteratively narrow the plaintext range
3. After ~1200 queries — flag recovered

```python
# core oracle call
def oracle(c):
    r = requests.post(URL, json={'ct': hex(c)})
    return r.json()['valid']  # True if padding is correct
```

---

*Placed in top 5% globally. Flag count: 11.*
