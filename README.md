# OxideOTP

**One-Time Pad File Encryption**

Based on [FinalCrypt](https://github.com/AcademySoftwareFoundation/finalcrypt) by George Veniatis
Rewritten and improved by Erick de S.C. Araújo

> **Note:** OxideOTP is distributed as a single standalone executable (`oxide-otp.exe`). No installer required — just run it directly. Source code is not publicly available.

---

## What is OxideOTP?

OxideOTP implements the **One-Time Pad (OTP)** — the only encryption algorithm mathematically proven to be unbreakable, when used correctly. Each byte of your file is XORed with a corresponding byte from a truly random key, producing ciphertext that reveals **zero information** about the original content.

```
Ciphertext = Plaintext ⊕ Key
Plaintext  = Ciphertext ⊕ Key
```

Unlike AES, RSA, or ChaCha20-based encryption, OTP security does not depend on computational hardness. Even with unlimited computing power, the ciphertext can never be cracked — provided the key is truly random, at least as long as the plaintext, and never reused.

---

## Quick Start

### 1. Generate Keys
```bash
oxide-otp keygen ./my_keys -s 1048576 -c 1
```

### 2. Encrypt
```bash
oxide-otp encrypt -k ./my_keys/key_1.bin secret.pdf
```
Result: `secret.pdf` → `secret.pdf.bit`

### 3. Decrypt
```bash
oxide-otp decrypt -k ./my_keys/key_1.bin secret.pdf.bit
```
Result: `secret.pdf.bit` → `secret.pdf`

---

## Technical Details

### XOR Engine — Bulk u64 Processing

The core encryption uses **8-byte (u64) bulk XOR** operations instead of byte-by-byte processing. This processes 8 bytes per CPU cycle, achieving near-linear throughput scaling.

```
Source:  [A0 A1 A2 A3 A4 A5 A6 A7] [A8 A9 ...]
Key:     [K0 K1 K2 K3 K4 K5 K6 K7] [K8 K9 ...]
         ──────── u64 XOR ────────  ─────────────
Result:  [R0 R1 R2 R3 R4 R5 R6 R7] [R8 R9 ...]
```

**Key features:**
- Bulk u64 XOR for 8 bytes/cycle throughput
- Zero-key-byte correction in MAC mode (0x00 → 0xFF) to prevent data leakage
- Automatic key wrapping when file exceeds key size
- Aligned and unaligned remainder handling

### Secure Random Number Generator (ChaCha20)

Key generation uses **ChaCha20** — a modern stream cipher designed by Daniel J. Bernstein. Combined with OS entropy and SHA-256 mixing, it produces cryptographically secure random bytes.

**Seed generation process:**

```
OS entropy (getrandom)     ──┐
                              ├─ XOR ── SHA-256 mix ── ChaCha20 seed
Nanosecond timestamp + seed ─┘
```

**Why ChaCha20?**
- 256-bit security level
- No hardware dependencies (unlike AES-NI)
- Excellent performance on all CPUs
- Deterministic output from seed (reproducible keys)

### MAC (Message Authentication Code)

OxideOTP uses a **MAC header** for file integrity verification.

**MAC structure (118 bytes):**
```
Offset 0-58:     Plaintext token (59 bytes)
                 "OxideOTP - One-Time Pad File Encryption - MAC Version 1.00"

Offset 59-117:   Encrypted token (59 bytes)
                 Plaintext ⊕ Key[0..59]
```

**How MAC works:**
1. During **encryption**: MAC header is prepended to the ciphertext
2. During **detection**: First bytes are compared against known plaintext tokens
3. During **verification**: Encrypted portion is XORed with key; result must match plaintext

**MAC versions supported:**
| Version | Plaintext Token | Status |
|---------|----------------|--------|
| V1 | `OxideOTP - File Encryption - Authentication Token` | Legacy |
| V2 | `OxideOTP - File Encryption - Auth Token Version 2` | Legacy |
| V3 | `OxideOTP - One-Time Pad File Encryption - MAC Version 1.00` | **Default** |

### Performance Characteristics

| Metric | Value |
|--------|-------|
| XOR throughput | ~8 bytes/cycle (u64) |
| Default buffer | 4 MB |
| Key generation | ~200 MB/s (ChaCha20) |
| Binary size | ~3.87 MB (release, LTO) |
| Startup time | ~50ms |

---

## CLI Commands

### Generate Keys
```bash
# Generate a single 1 MB key
oxide-otp keygen ./my_keys -s 1048576 -c 1

# Generate 10 keys of 10 MB each
oxide-otp keygen ./my_keys -s 10485760 -c 10

# Generate with extra entropy scrambling
oxide-otp keygen ./my_keys -s 1048576 -c 1 --scramble
```

### Verify Key
```bash
# Check a single key
oxide-otp check ./my_keys/key_1.bin

# Check key against source files
oxide-otp check ./my_keys/key_1.bin -s file1.txt file2.pdf
```

### Encrypt Files
```bash
# Encrypt a single file
oxide-otp encrypt -k ./my_keys/key_1.bin secret_document.pdf

# Encrypt multiple files
oxide-otp encrypt -k ./my_keys/key_1.bin file1.txt file2.pdf image.png

# Encrypt with custom output directory
oxide-otp encrypt -k ./my_keys/key_1.bin secret.pdf -o ./encrypted/

# Encrypt with password (additional security layer)
oxide-otp encrypt -k ./my_keys/key_1.bin secret.pdf -p "my_strong_password"

# Encrypt entire directory
oxide-otp encrypt -k ./my_keys/key_1.bin ./my_documents/

# Verbose mode
oxide-otp encrypt -k ./my_keys/key_1.bin secret.pdf -v
```

### Decrypt Files
```bash
# Decrypt a single file
oxide-otp decrypt -k ./my_keys/key_1.bin secret.pdf.bit

# Decrypt multiple files
oxide-otp decrypt -k ./my_keys/key_1.bin file1.txt.bit file2.pdf.bit

# Decrypt with password (if used during encryption)
oxide-otp decrypt -k ./my_keys/key_1.bin secret.pdf.bit -p "my_strong_password"
```

### File Info
```bash
oxide-otp info secret.pdf secret.pdf.bit
```

---

## GUI Usage

Double-click `oxide-otp.exe` (or run without arguments) to launch the graphical interface.

### Encrypt Tab
1. Click **Add Files** or **Add Directory** to select files
2. Click **Select Key File** or **Select Key Directory** for your key
3. (Optional) Set output directory and password
4. Click **ENCRYPT**

### Decrypt Tab
1. Click **Add .bit Files** to select encrypted files
2. Click **Select Key** for the matching key
3. (Optional) Enter password if one was used
4. Click **DECRYPT**

### Key Generator Tab
1. Set key size (default: 1 MB)
2. Set number of keys
3. Choose output directory
4. Click **GENERATE KEYS**

### Settings
- **Disable MAC** — Removes MAC header (faster, but no integrity check)
- **Buffer size** — 1 MB / 4 MB / 16 MB (larger = faster for big files)

---

## Security Considerations

### OTP Rules (Critical)
1. **Never reuse a key** — Each key encrypts exactly one file
2. **Key must be truly random** — Generated with cryptographic RNG (ChaCha20)
3. **Key must be at least as long as the plaintext** — For maximum security
4. **Key must be kept secret** — Store securely, destroy after use

### What Makes OxideOTP Secure?
- **No mathematical structure** — XOR with random key produces uniform output
- **No key derivation** — Key is used directly, no password stretching
- **No mode of operation** — Each byte is independent
- **Verifiable** — MAC header confirms correct key usage

### Threat Model
| Attack | Protection |
|--------|------------|
| Brute force | Impossible — all keys produce valid plaintext |
| Frequency analysis | Failed — output is uniform random |
| Known plaintext | Failed — key is truly random |
| Quantum computing | Immune — OTP is information-theoretically secure |

---

## License

MIT License

## Credits

- **FinalCrypt** by George Veniatis — Original Java implementation
- **OxideOTP** by Erick de S.C. Araújo — Rust rewrite with performance improvements
