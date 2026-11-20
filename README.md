# Blocking `smartctl` + Logical Unit ID on an RTL9210B USB–NVMe Bridge

A reproducible guide to patching Realtek **RTL9210B** bridge firmware so that:

- `smartctl -d sntrealtek` can no longer read **SMART / health / NVMe identity / EUI‑64**, and
- `smartctl -x -d scsi` can no longer read the **SCSI Logical Unit ID** (the VPD page 0x83 NAA
  identifier),

while the disk keeps working normally for ordinary reads and writes.

**Contributors**

- **Codex** (OpenAI) — original reverse‑engineering: firmware layout, integrity fields,
  SCSI command‑path analysis, and the first experiments.
- **Claude** (Anthropic — Claude Code) — located the MIPS16e `E4` dispatchers and the VPD‑0x83
  responder, built and verified the patches on hardware, and wrote this guide.

> **The releases below contain everything you need — the flashing tool and the pre‑built
> firmware.** For most people this is just: download the release for your enclosure, flash the
> `.bin` with the included tool, done. You only need the rest of this guide if you want to
> understand the patch or build it for a firmware version that isn't pre‑built.

**Tested enclosures** (both RTL9210B‑based):

| Enclosure | Firmware | Release (tool + firmware) | SMART block | LU‑ID block |
| --- | --- | --- | --- | --- |
| **Sabrent EC‑SNVE** ([Amazon](https://www.amazon.com/dp/B08RVC6F9Y)) | 1.35.6 | [Download](https://github.com/GoofyNest/RTL9210B-SMARTCTL/releases/tag/1) | ✅ Verified | ✅ Verified |
| **ELUTENG Transparent M.2 NVMe SSD** ([Amazon](https://www.amazon.com/dp/B083XWBR2V)) | 1.33.5 | [Download](https://github.com/GoofyNest/RTL9210B-SMARTCTL/releases/tag/11) | ✅ Verified | ⚠️ Built, flash‑test recommended |

Flash the image that matches **your** enclosure's firmware version. Any RTL9210B bridge should
work — the patch targets the firmware, not the enclosure — but the two above are the ones built
and tested. The LU‑ID patch is confirmed on the Sabrent (1.35.6); the ELUTENG (1.33.5) LU‑ID
byte was located by the same method and is included in its image, but hasn't been flash‑tested
on that hardware yet — flash it with a CH341A ready.

---

## ⚠️ Read this first

1. **A Realtek vendor SCSI command sent to the wrong drive can crash it — including a
   system/boot disk.** Before any read or flash, confirm which disk is your RTL9210B bridge
   (see [step 4](#4-identify-your-bridge)) and never target anything else.
2. **Flashing can brick the device.** Have a **CH341A SPI programmer** ready before you start
   so you can restore stock firmware (see [Unbricking](#8-unbricking)).
3. **Only edit a copy.** Never modify your golden/recovery image or your only known‑good
   stock `.bin`.
4. **Match the firmware version.** A 1.35.6 patch must be built from a 1.35.6 image, a 1.33.5
   patch from a 1.33.5 image — the byte offsets differ between versions.

---

## 1. What the patches do

### 1a. The SMART block (`0xE4` → `0xE7`)

`smartctl -d sntrealtek` does **not** use ordinary disk commands. It tunnels every NVMe admin
read through a single **Realtek vendor SCSI command** whose CDB byte 0 is `0xE4`:

```text
Byte:   00 01 02 03 04 05 ...
Value:  E4 00 10 06 00 00 ...
        ^^       ^^ ^^
        |        |  +-- CNS (00 = Identify Namespace → carries EUI‑64 at resp 0x78–0x7F)
        |        +----- NVMe admin opcode (06 = Identify, 02 = Get Log Page = SMART/health)
        +-------------- Realtek vendor command 0xE4
```

So **everything** smartctl reports this way — identity, serial, model, firmware revision,
SMART/health, error log, self‑test log, and EUI‑64 — arrives over opcode `0xE4`. Ordinary block
I/O uses standard SCSI/UASP opcodes (`READ(10/16)`, `WRITE(10/16)`, …), **not** `0xE4`.

The firmware recognizes the `0xE4` opcode inside its SCSI command dispatchers with a
**MIPS16e `CMPI r, 0xE4`** (compare‑immediate) instruction. The patch changes that immediate
from **`0xE4` to `0xE7`**:

- A real `0xE4` request no longer matches any recognizer → it falls through to the default
  **"unsupported command" → CHECK CONDITION / ILLEGAL REQUEST (sense 5 / 0x20 / 0x00)**.
- smartctl's Identify and Get‑Log‑Page both fail → **no SMART, no health, no identity, no
  EUI‑64**.
- `0xE7` is an unused opcode, so nothing else starts matching it.
- Standard read/write opcodes are untouched → **the disk still works**.
- The enclosure's own on‑screen thermal/health monitor uses a separate on‑chip ROM path and
  is **unaffected**.

### 1b. The Logical Unit ID block (VPD 0x83 designator length → 0)

Separately, `smartctl -x -d scsi` reports a **Logical Unit id** (e.g.
`0x3102020126482807`) — a stable NAA identifier. This does **not** go through the `0xE4` path;
it is a standard SCSI **INQUIRY → EVPD → page 0x83** response. The RTL9210B does **not** pass a
drive VPD page through — NVMe drives have no SCSI VPD pages — it **synthesizes** the page in
firmware, building the designator and filling the value from the attached NVMe drive's identity
data.

The firmware's live VPD‑0x83 responder builds the NAA designator header one byte at a time:

```text
descriptor:  01   03   00   08   <8 identifier bytes>
             ^^   ^^        ^^
             |    |         +-- designator LENGTH = 8   ← the patch zeroes this
             |    +------------ designator type = NAA
             +----------------- code set = binary
```

The patch sets the **designator length from `0x08` to `0x00`**. With a zero‑length NAA
designator the bridge no longer returns a valid identifier, so **smartctl can no longer read a
Logical Unit id** — it reports a length/parse error in place of the value, and the real
identifier is never disclosed. Ordinary operation is unaffected; the disk still enumerates and
reads/writes normally. (Note: this is the minimal change; the page is technically malformed
rather than cleanly empty, which Windows tolerates. Flash‑test on your OS with a CH341A ready.)

Both patches are tiny, reversible immediate edits that keep the dispatcher structure intact.

---

## 2. Firmware image layout

The image is **409,600 bytes (`0x64000`)**:

| Region | Range | Contents |
| --- | --- | --- |
| Header | `0x0000`–`0x3FFF` | magic `0x97463136` @`0x00`, integrity fields, version |
| Payload | `0x4000`–`0x63FFF` | MIPS32r2 + MIPS16e code, mapped at runtime `0x8C000000 + fileOffset` |

Integrity fields that **must be regenerated** after any payload edit:

| Header offset | Field | Computed over |
| --- | --- | --- |
| `0x10`–`0x13` | **CRC‑32** of payload (IEEE poly `0xEDB88320`, reflected), little‑endian | file `0x4000`–`0x63FFF` |
| `0x64`–`0x83` | **SHA‑256** of payload (32 bytes) | the same range |
| `0x4C` / `0x50` / `0x54` | version major / minor / patch | leave unchanged |

If you change payload bytes but leave the CRC/SHA stale, the image is inconsistent and the MP
tool's CRC check may reject it. The build script in [step 5](#5-build-the-patched-firmware)
always regenerates both.

---

## 3. What you need

**If you're using a pre‑built release (most people):**

- **Windows** (the flashing tool is Windows‑only; it does not work well under Wine or in a VM).
- The **release for your enclosure** (links above) — it includes the flashing tool
  (UTHSB_MPtool) and the patched firmware image.
- A **CH341A SPI programmer** (recovery insurance — strongly recommended).
- `smartctl` 7.5+ (smartmontools) to verify the result.

**If you're building your own** (a firmware version that isn't pre‑built): additionally a
**stock image** matching your bridge's version, your device's **`.cfg`**, and a
**golden/recovery** image — all also in the releases for the tested versions.

---

## 4. Identify your bridge

Run this (non‑destructive) and note which disk is your RTL9210B — the one with
`BusType = USB` whose model/size matches the NVMe inside your enclosure:

```powershell
Get-Disk | Select-Object Number, FriendlyName, BusType, @{n='SizeGB';e={[math]::Round($_.Size/1GB)}}
```

The examples below use `/dev/pd1` — **change the number to match your device**, and never
point any vendor command at your system/boot disk.

---

## 5. Build the patched firmware

*(Skip this if you're flashing a pre‑built release image.)* The patch is a handful of one‑byte
edits plus regenerated checksums.

### 5a. Patch offsets by version

**Firmware 1.35.6** (Sabrent EC‑SNVE):

| Purpose | File offset | Change |
| --- | --- | --- |
| SMART block (`CMPI 0xE4`) | `0x0EB7A` | `E4 → E7` |
| SMART block | `0x0ECCE` | `E4 → E7` |
| SMART block | `0x132C0` | `E4 → E7` |
| SMART block | `0x14646` | `E4 → E7` |
| SMART block | `0x16B68` | `E4 → E7` |
| **LU‑ID block** (VPD‑0x83 designator length, handler `0x0D316`) | `0x0D4DA` | `08 → 00` |

**Firmware 1.33.5** (ELUTENG Transparent M.2):

| Purpose | File offset | Change |
| --- | --- | --- |
| SMART block (`CMPI 0xE4`) | `0x0C334` | `E4 → E7` |
| SMART block | `0x1075C` | `E4 → E7` |
| SMART block | `0x11932` | `E4 → E7` |
| SMART block | `0x13258` | `E4 → E7` |
| **LU‑ID block** (VPD‑0x83 designator length, handler `0x0B024`) | `0x0B17E` | `08 → 00` |

For any other firmware version, locate the SMART sites with the scan in
[step 9](#9-finding-the-sites-on-other-firmware-versions), and the LU‑ID site by the method in
the same step.

### 5b. Build script (PowerShell)

Save your stock image as `stock.bin` next to this script, then run:

```powershell
# --- Build an RTL9210B image: SMART block (+ optional LU-ID block) -------
$In  = '.\stock.bin'          # stock image, version-matched
$Out = '.\patched.bin'

# Pick the offsets for YOUR firmware version (see 5a):
$e4Sites = @(0x0C334, 0x1075C, 0x11932, 0x13258)             # 1.33.5 SMART sites
$luSite  = 0x0B17E                                           # 1.33.5 LU-ID length byte (0 to skip)
# $e4Sites = @(0x0EB7A, 0x0ECCE, 0x132C0, 0x14646, 0x16B68) # 1.35.6 SMART sites
# $luSite  = 0x0D4DA                                         # 1.35.6 LU-ID length byte

$payOff = 0x4000; $payLen = 0x60000; $size = 0x64000

Add-Type -TypeDefinition 'using System; public static class CrcE4 {
  public static uint Calc(byte[] b,int o,int l){ uint c=0xffffffffu;
    for(int i=o;i<o+l;i++){ uint v=c^b[i]; for(int k=0;k<8;k++) v=((v&1)!=0)?((v>>1)^0xedb88320u):(v>>1); c=v; } return ~c; } }'

$b = [IO.File]::ReadAllBytes($In)
if ($b.Length -ne $size) { throw "unexpected size $($b.Length)" }
foreach ($s in $e4Sites) { if ($b[$s] -ne 0xE4) { throw ("SMART site 0x{0:X5} is 0x{1:X2}, expected 0xE4 (wrong firmware version?)" -f $s,$b[$s]) } }
if ($luSite -ne 0 -and $b[$luSite] -ne 0x08) { throw ("LU-ID site 0x{0:X5} is 0x{1:X2}, expected 0x08" -f $luSite,$b[$luSite]) }

$p = [byte[]]$b.Clone()
foreach ($s in $e4Sites) { $p[$s] = 0xE7 }                   # E4 -> E7  (SMART block)
if ($luSite -ne 0) { $p[$luSite] = 0x00 }                    # 08 -> 00  (LU-ID block)

$crc = [CrcE4]::Calc($p,$payOff,$payLen)                     # regenerate CRC-32
[Array]::Copy([BitConverter]::GetBytes($crc),0,$p,0x10,4)
$sha = [Security.Cryptography.SHA256]::Create()
try { $h = $sha.ComputeHash($p[$payOff..($payOff+$payLen-1)]) } finally { $sha.Dispose() }
[Array]::Copy($h,0,$p,0x64,32)                               # regenerate SHA-256
[IO.File]::WriteAllBytes($Out,$p)

# self-consistency check
$ok = ([BitConverter]::ToUInt32($p,0x10) -eq [CrcE4]::Calc($p,$payOff,$payLen))
[pscustomobject]@{
  Output   = $Out
  Version  = ('{0}.{1}.{2}' -f $p[0x4c],$p[0x50],$p[0x54])
  SmartSitesPatched = $e4Sites.Count
  LUIDPatched = ($luSite -ne 0)
  PayloadCRC   = ('0x{0:X8}' -f $crc)
  CRC_SelfConsistent = $ok
  SHA256   = (Get-FileHash -LiteralPath $Out -Algorithm SHA256).Hash
}
```

> **PowerShell 5.1 notes:** use `[Security.Cryptography.SHA256]::Create().ComputeHash(...)`
> (there is no `::HashData`), and format hex with `'{0:X2}' -f $_` (there is no
> `[Convert]::ToHexString`).

### 5c. Optional — change the serial

The NVMe serial is a fixed‑width ASCII field in the identity block (its exact offset is
version/config‑specific). To set your own serial, overwrite that field with an ASCII string
padded to its width **before** the CRC/SHA step, then let the script regenerate the checksums.
This is independent of the two blocks above.

---

## 6. Flash the patched firmware (UTHSB_MPtool, included in the release)

1. Back up: dump your current device **configuration** with the MP tool and save it.
2. Set the firmware file to your patched `.bin`, set the golden/recovery image, and select
   your device `.cfg`. (The release ships the tool and the matching support files.)
3. (Recommended) In the `.cfg`, set `SERIAL` to match your dump if defined, and comment out
   `DISK_IPS_THRES` with `;`.
4. Connect the enclosure, click the flash/update button, and wait for it to finish.
5. **Replug** the enclosure so it re‑enumerates on the new firmware.

> Flashing writes only the flash payload (both mirrored banks); it cannot change on‑chip mask
> ROM. That's fine — both the `E4` dispatchers and the VPD‑0x83 responder patched here live in
> the flash payload.

For the full flashing walkthrough (screenshots, config‑file reference, per‑device notes) and
recovery, see the upstream project this guide builds on:
**https://github.com/bensuperpc/RTL9210**

---

## 7. Verify

**SMART block** — point smartctl at the bridge (adjust `pd1`):

```powershell
smartctl -d sntrealtek -a /dev/pd1
```

**Success looks like a failure** — smartctl should be rejected at the first Identify:

```text
status=2: sense_key=5 asc=20 ascq=0        (ILLEGAL REQUEST / INVALID COMMAND OPCODE)
sntrealtek_device::nvme_pass_through: scsi error: unsupported scsi opcode
Read NVMe Identify Controller failed: scsi error unsupported scsi opcode
```

**LU‑ID block** — check the standard SCSI identity page:

```powershell
smartctl -x -d scsi /dev/pd1
```

The **`Logical Unit id:`** line should no longer show a real NAA value — it reports a
length/parse error (e.g. `error: NAA 3 or 5 length ...`) instead, i.e. the identifier is not
disclosed.

Confirm the disk is still healthy and usable:

```powershell
Get-Disk | Where-Object BusType -eq 'USB' | Select-Object Number, HealthStatus, OperationalStatus
Get-Volume          # your volume should still mount
smartctl --scan     # device still enumerates over standard SCSI
```

Add `-r ioctl,2` to the `sntrealtek` command to see the raw CDBs — every NVMe transaction's CDB
starts with `e4 …` and comes back rejected.

---

## 8. Unbricking

If a flash fails and the bridge no longer enumerates, follow the **Unbrick** guide in the
upstream project — it covers opening the enclosure, identifying the SPI flash chip, and
reflashing stock with a CH341A SPI programmer:

**→ https://github.com/bensuperpc/RTL9210#unbrick**

Keep an unmodified stock `.bin` on hand for exactly this.

---

## 9. Finding the sites on other firmware versions

To reproduce the offsets on a version not listed above:

### SMART (`0xE4`) sites

1. **Confirm the host side.** `smartctl -r ioctl,2 -d sntrealtek …` prints every 16‑byte CDB.
   Byte 0 is `e4` for all NVMe admin reads — Identify (`e4 00 10 06 …`) and Get Log Page
   (`e4 00 02 02 …`). This is the single opcode to block.
2. **Scan for MIPS16e, not MIPS32.** This firmware's command dispatch is MIPS16e; a MIPS32
   immediate scan finds nothing. Scan even file offsets for low byte `0xE4` with a high byte
   in `0x70`–`0x77` (the `CMPI` encoding):

   ```powershell
   $b = [IO.File]::ReadAllBytes('.\stock.bin')
   for ($i = 0x4000; $i -lt 0x63FFF; $i += 2) {
     if ($b[$i] -eq 0xE4 -and $b[$i+1] -ge 0x70 -and $b[$i+1] -le 0x77) {
       '0x{0:X5}  (runtime 0x{1:X8})  CMPI r{2},0xE4' -f $i, (0x8C000000 + $i), ($b[$i+1] -band 7)
     }
   }
   ```

3. **Confirm each hit is a real dispatcher.** The genuine `CMPI 0xE4` sites are surrounded by
   other SCSI opcode compares — `INQUIRY 0x12`, `READ10 0x28`, `WRITE10 0x2A`, `READ16 0x88`,
   `WRITE16 0x8A`, etc. Keep the ones embedded in such a run. There are several because the
   bridge has a separate dispatch chain per transport (BOT / UASP / CD‑ROM LUN).

### LU‑ID (VPD‑0x83) site

1. **Find the VPD page‑code switches.** Scan for `CMPI r,0x83` (low byte `0x83`, high byte
   `0x70`–`0x77`). The firmware has **two** full VPD dispatchers (one per transport); each is a
   tight cluster that also compares `0x80`, `0xB0`, `0xB1`, `0xB2` (the other supported VPD
   pages) within ~0x120 bytes. Both 1.35.6 and 1.33.5 follow this pattern.
2. **Pick the live one.** The dispatcher that sits *near the `0xE4` region* is the inactive
   twin; the **other** cluster is the one your host actually reads (1.35.6: `0x0D316` live vs
   `0x16E8C` twin; 1.33.5: `0x0B024` live vs `0x10A48` twin). If unsure, flash‑test — patching
   the wrong one has no effect, which is itself diagnostic.
3. **Find the designator length.** In the live handler's window, the NAA descriptor header is
   built with `LI r,0x01` (code set) → `LI r,0x03` (NAA type) → `LI r,0x08` (designator
   length). The length `LI` is the byte to flip to `0x00` (its low byte is `0x08`, high byte
   `0x68`–`0x6F`).
4. Patch those offsets, regenerate the checksums, flash, and verify as in steps 5–7.

---

*This procedure modifies device firmware and is provided as‑is. Flashing can brick hardware;
proceed at your own risk and keep a stock image plus a CH341A programmer ready.*
