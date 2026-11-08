# CSI on the ath10k/ath10k-ct platform: what we found

**Author:** Claude (Anthropic LLM), working interactively with a human operator.
**Scope:** Linksys EA8300 (Qualcomm IPQ4019 host SoC + QCA9888 PCIe radio +
QCA4019 integrated radios), OpenWrt 24.10.8, `ath10k-ct` firmware
`10.4b-ct-9888-fW-13-5ae337bb1` (Candela Technologies build, dated ~2020).

## Table of contents

1. [Goal](#1-goal)
2. [The mechanism: `ct_special` and the CSI WMI event](#2-the-mechanism-ct_special-and-the-csi-wmi-event)
3. [Live hardware testing](#3-live-hardware-testing)
4. [Decoding the buffer: it's metadata, not channel data](#4-decoding-the-buffer-its-metadata-not-channel-data)
5. [Prior art: this is a known, decade-old, unsolved problem](#5-prior-art-this-is-a-known-decade-old-unsolved-problem)
6. [Firmware teardown](#6-firmware-teardown)
7. [Identifying the CPU architecture: Xtensa](#7-identifying-the-cpu-architecture-xtensa)
8. [Ghidra analysis and the `ct_special` dispatcher function](#8-ghidra-analysis-and-the-ct_special-dispatcher-function)
9. [Where this stands / the open question](#9-where-this-stands--the-open-question)
10. [Next steps](#10-next-steps)
11. [Practical methodology notes](#11-practical-methodology-notes)

---

## 1. Goal

Four EA8300 routers running OpenWrt in an 802.11s mesh (one gateway/DHCP
router + three APs) were investigated as a possible platform for **WiFi
CSI-based sensing** — using radio signal measurements to detect human
presence or movement without a worn device. CSI (Channel State Information)
measures how a received signal's amplitude and phase vary across OFDM
subcarriers; a person moving through the signal path perturbs that pattern
in ways RSSI alone can't capture.

The question going in: can this *specific, already-owned* hardware do this,
or is different hardware (ESP32-CSI, Nexmon/Broadcom) required?

## 2. The mechanism: `ct_special` and the CSI WMI event

The routers run **`ath10k-ct`**, Candela Technologies' fork of the `ath10k`
driver/firmware, not stock `ath10k`. Two components are involved, and it's
important to keep them distinct:

- **The kernel driver** (`ath10k-ct`, open source, GPL,
  [github.com/greearb/ath10k-ct](https://github.com/greearb/ath10k-ct)) —
  runs on the host SoC (IPQ4019, ARM, Linux/OpenWrt). Fully inspectable.
- **The radio firmware** (`firmware-5.bin`) — runs on a separate embedded
  CPU core *inside* the QCA9888/QCA4019 chip itself. Closed-source binary
  blob. The driver can send it commands and receive events from it, but has
  no visibility into what it does internally.

The driver exposes a debugfs file per radio (phy):
`/sys/kernel/debug/ieee80211/phyN/ath10k/ct_special`. Writing a specific
encoded 64-bit value to this file sends a `WMI_PDEV_SET_SPECIAL` command to
the firmware. The file's own built-in documentation (`cat` it with no
arguments) lists dozens of IDs; the one relevant here:

```
id: 0xD Enable CSI reporting for at least probe requests.
```

Encoding: `value = (id << 32) | val`. So enabling CSI reporting on a given
radio is:

```sh
echo $(( (0xD << 32) | 1 )) > /sys/kernel/debug/ieee80211/phy2/ath10k/ct_special
```

This issues `ath10k_wmi_pdev_set_special(ar, SET_SPECIAL_ID_CSI, 1)` in the
driver, confirmed live by a kernel log line:

```
ath10k_pci 0000:01:00.0: Setting CT-CSI dump override to 0x1
```

When the firmware later emits a `WMI_10_4_CSI_MESG_EVENTID` event, the
driver's handler (`ath10k_wmi_event_csi_mesg` in `wmi.c`) just dumps the raw
event bytes straight to the kernel log:

```c
int ath10k_wmi_event_csi_mesg(struct ath10k *ar, struct sk_buff *skb)
{
    ...
    dev_printk(lvl, ar->dev, "ath10k_pci ATH10K_CSI_BUFFER:\n");
    while (q < len) {
        printk("%sath10k: [%04d]: %08X %08X ...\n", ...);
        ...
    }
    dev_printk(lvl, ar->dev, "ATH10K_END\n");
    return 0;
}
```

No parsing, no structure — whatever the firmware sends appears verbatim in
`dmesg` as raw hex words. This matters a lot for what follows: the driver is
not hiding or discarding anything. What you see in `dmesg` is exactly what
the firmware chose to send.

## 3. Live hardware testing

All testing used the Ansible-managed fleet's existing SSH key, with a
specific safety protocol derived from the mesh's actual network topology
(important because the test client's own path to the internet ran through
one specific mesh hop — testing was done on *other* nodes not in that path,
to guarantee the operator never lost connectivity regardless of outcome).

Three different trigger conditions were tested, on two different radios
(the idle 802.11s mesh backhaul radio, and a radio actively serving real
client traffic):

1. **Ambient traffic** — just enabling `ct_special` id `0xD` on a radio with
   real, already-associated clients. Result: CSI events fired periodically,
   each tracking a distinct client's own background WiFi scan behavior
   (confirmed — see below).
2. **A directed probe from a controlled device** — using `nmcli device wifi
   rescan ... ssid "..."` from an otherwise-idle WiFi radio, targeted at the
   AP's SSID/frequency, with no association. This *did* trigger a CSI event
   (on an AP-mode radio; it did **not** work on a bare 802.11s mesh-point
   radio in the same test).
3. **An explicit forced TXBF sounding exchange** — `ct_special` id `0x1004`
   ("set rc-txbf-probe"), which forces the AP to send an explicit 802.11ac
   beamforming sounding frame to its currently-associated, TXBF-capable
   clients right now, rather than waiting for one to happen naturally.

**All three trigger methods produced the identical buffer format and size.**
This is the key empirical result of the live-testing phase: the firmware's
behavior does not change based on *how* a packet exchange was induced.

Stability throughout: at no point did any router crash, reboot, or lose
network connectivity. Every `ct_special` write returned immediately and was
acknowledged by a kernel log line; uptime counters never reset.

## 4. Decoding the buffer: it's metadata, not channel data

Every captured `ATH10K_CSI_BUFFER` event was **exactly 13 32-bit words (52
bytes)**, regardless of trigger method, client, or radio. A representative
capture:

```
[0000]: 00000001 54525453 4452544E 00000000 FC3DE3E4 00003407 444E4544 54525453
[0008]: 46445254 00000001 FC3DE3E4 00003407 444E4544
```

Decoding word-by-word, this structure was fully accounted for:

| Words | Content |
|---|---|
| 0 | Monotonic sequence counter (global, increments across *all* clients) |
| 1–2 | ASCII framing tag (`"STRT"` + a second marker) |
| 3 | Sequence counter again |
| **4–5** | **The triggering client's own MAC address**, little-endian, zero-padded to 8 bytes |
| 6 | ASCII framing tag (`"DEND"`) |
| 7–12 | The same structure repeats a second sub-record |

The MAC-address decode was verified directly: a test device's own known MAC
(`34:cf:f6:c5:0e:54`) was found byte-for-byte in a captured buffer as
`C5F6CF34 0000540E`. Timing analysis across ~15 distinct observed client MAC
addresses over 30+ minutes showed each one firing on a clean, fixed period
matching normal OS background-scan intervals (roughly 20s, 120s, 160s, or
540s depending on device) — strong independent confirmation that this event
fires on ordinary client probe/scan behavior, and that every word in the
buffer is accounted for as bookkeeping (sequence + MAC + framing), with
**no remaining space for actual amplitude/phase channel measurements.**

This was cross-checked against the one serious prior attempt to decode this
exact buffer (see [Section 5](#5-prior-art-this-is-a-known-decade-old-unsolved-problem)) — that attempt's author also could not get a
consistent amplitude/phase interpretation, which makes sense in retrospect:
there's no real channel data in there to decode.

## 5. Prior art: this is a known, decade-old, unsolved problem

This was not a dead end specific to this session's hardware or methodology.
Four independent public threads, spanning 2016 to the present, all hit the
identical wall:

- **2016, mainline `ath10k` mailing list** — someone asked about CSI
  extraction. The actual `ath10k` maintainer, Kalle Valo, replied: *"This is
  the first time I'm even hearing about CSI so I can't really help much,
  sorry"* — and noted the firmware doesn't report it.
  ([thread](http://lists.infradead.org/pipermail/ath10k/2016-August/008163.html))
- **2020, `ath10k-ct` GitHub issue #122** — titled *"Add firmware option to
  ignore 'not sounding bit' on RX packets — return CSI info for all RX, not
  just sounding packets."* Explains the real, narrow trigger condition:
  firmware only reports CSI for NDP (Null Data Packet) explicit-sounding
  frames with a specific bit cleared — not ordinary traffic. One commenter
  (the same person behind the PR below) tried decoding the real payload and
  called the result "not really satisfactory." The fork's own maintainer
  replied: *"I am unlikely to work on this anytime soon... I don't know the
  magic incantations in the firmware to do so."* Still open, unresolved.
  ([issue](https://github.com/greearb/ath10k-ct/issues/122))
- **`ath10k-ct` PR #98, "csi over relayfs"** — the same person who posted the
  inconclusive decode wrote a real accumulator + relayfs transport for CSI
  data (targeting an older `ath10k-5.2` driver tree). PR body: *"I have
  implemented the code that transports the CSI data over relayfs. I have
  also played around and tried to decode it, however, so far without any
  success."* Never merged.
  ([PR](https://github.com/greearb/ath10k-ct/pull/98))
- **A separate researcher's unanswered inquiry**, and **an OpenWrt forum
  thread** asking the identical question about QCA9984 — both went
  unanswered.

Four different people, three different venues, nine years apart, zero
successes. The person with the most plausible access to deeper information —
the fork's own maintainer — gave the same "I don't know" answer as the
original mainline maintainer did four years earlier.

## 6. Firmware teardown

`firmware-5.bin` (the file OpenWrt loads onto the QCA9888) uses a documented,
open container format (`ATH10K_FIRMWARE_MAGIC = "QCA-ATH10K"`), defined in
the open driver's `hw.h`:

```c
struct ath10k_fw_ie {
    __le32 id;
    __le32 len;
    u8 data[0];
};
```

Parsing it (magic string, padded to 4 bytes, then repeated
`{id, len, data}` entries, each padded to a 4-byte boundary) cleanly yields
several IEs, including:

- `ATH10K_FW_IE_FW_VERSION` (id 0) — a plain ASCII string, verified to match
  exactly what the live router reports (`10.4b-ct-9888-fW-13-5ae337bb1`).
  This confirmed the parser was correct before going any further.
- `ATH10K_FW_IE_FW_IMAGE` (id 3) — the main executable payload, 375,210
  bytes.
- `ATH10K_FW_IE_FW_CODE_SWAP_IMAGE` (id 7) — a second, auxiliary executable
  payload, 246,425 bytes — loaded into a separate memory region, called on
  from the main image (confirmed in [Section 7](#7-identifying-the-cpu-architecture-xtensa)).
- `ATH10K_FW_IE_OTP_IMAGE` (id 4) — a small calibration/OTP-related blob,
  8,990 bytes.

Inside the main `FW_IMAGE` payload, right after its own small header
(magic `"SGMT"`, a count field, and a declared load address/length), the
actual code begins. This inner `SGMT` structure is **not** documented in the
open driver — it's internal to how the firmware's own loader organizes
itself, and had to be inferred empirically (see next section).

**Deliberately not included in this repo:** the firmware binary itself. It's
Qualcomm/Candela's proprietary property; this document describes how to
extract and parse it from hardware you own, not a copy of it.

## 7. Identifying the CPU architecture: Xtensa

The firmware's embedded CPU architecture is undocumented publicly for this
exact chip. This section is the most methodologically interesting part of
the investigation, including the dead ends, because the process (and
exactly how to validate against false positives) generalizes to similar
unknown-firmware problems.

### 7.1 The clue

The previous-generation Atheros WiFi chip family, `ath9k`, is confirmed via
public discussion to run its firmware on a **Tensilica Xtensa** core (the
same CPU family used in the ESP32). Chip vendors commonly reuse an embedded
core across successive product generations to avoid re-tooling their
firmware compiler — a reasonable hypothesis to test, not an assumption to
rely on.

### 7.2 Why naive heuristics produced false positives

A first-pass heuristic — disassemble with each candidate architecture, see
how many consecutive instructions decode before hitting an invalid opcode —
pointed at **ARM Thumb**, scoring highest of ~14 architectures tested via
Capstone. Visual inspection of the actual output showed why this was
misleading: Thumb's dense 16-bit instruction encoding means *almost any*
byte stream decodes to *something* syntactically valid (lots of coincidental
`movs r0, r0` no-ops), without ever hitting bytes it can't interpret as an
instruction. A high "doesn't crash" score is not the same as "is actually
correct."

### 7.3 A better metric: branch-target self-consistency

The metric that actually discriminates correct decoding from coincidence:
for every branch/call/jump instruction found, does its target address land
exactly on the start of another decoded instruction elsewhere in the file?
Real compiled code's control-flow edges always land on real instruction
boundaries; decoding garbage as the wrong architecture produces targets that
scatter essentially randomly.

Using real Xtensa disassemblers (Espressif's `xtensa-esp32-elf-objdump` and
the Debian `xtensa-lx106-elf-objdump` package — both installable without
root via `apt-get download` + `dpkg-deb -x`, no `sudo` needed), a brute-force
sweep across the *entire* firmware image, for every architecture available
(14 via Capstone, 2 Xtensa variants, plus ARC via
`binutils-arc-linux-gnu`), scoring 4KB windows every 4KB across the whole
file, found:

```
Xtensa-LX106   64.5%  offset=94208  branches=220
Xtensa-ESP32   57.6%  offset=94208  branches=250
(every other architecture: 0% or no qualifying window)
```

Two independently-built Xtensa toolchains agreeing on the exact same offset,
while every other architecture found nothing, is strong evidence on its
own. Refining further and mapping consistency across the whole combined
image (main image + swap image, placed contiguously) found multiple 4KB
windows scoring **97–100%** branch-target consistency — hundreds of branches,
every single one landing on a real instruction boundary. That level of
self-consistency does not happen by chance.

### 7.4 Confirming the two-segment memory layout

A second, independent cross-check: several `call0` instructions in the
confirmed-good region had targets that fell *outside* the main image's own
address range (`0x40a000`–`0x4659aa`). Computing where the `FW_CODE_SWAP_IMAGE`
segment would sit if loaded immediately after the main image
(`0x4659aa`–`0x4a1c43`) showed **every one** of those "out of range" targets
landing cleanly inside that hypothesized swap region. This is exactly what
real code calling into a real auxiliary code segment looks like, and it
let a later Ghidra import treat the two segments as one coherent combined
memory image for proper cross-reference analysis.

## 8. Ghidra analysis and the `ct_special` dispatcher function

**Ghidra 12.2 has native Xtensa support built in** — no separate processor
extension needed (older community Xtensa modules for Ghidra explicitly note
they're redundant on 11.0+). Setup notes (all without `sudo`):

- Ghidra itself: `gh release download` from
  `NationalSecurityAgency/ghidra`, a self-contained zip.
- Requires **Java 25** specifically (not 21) — check
  `Ghidra/application.properties` → `application.java.min`. Grabbed via
  `apt-get download openjdk-25-jdk-headless openjdk-25-jre-headless` and
  `dpkg-deb -x` into a local directory (both packages are needed; `java`
  itself ships in the `jre` package, not the `jdk` one).
  A handful of `/etc`-absolute symlinks inside the extracted JDK (security
  policy files, `cacerts`) need to be replaced with real copies from the
  same extracted tree, since there's no permission to write to the real
  `/etc`.
- Run via `support/analyzeHeadless` (no GUI/display needed), importing the
  combined main+swap binary as a **Raw Binary**, language
  `Xtensa:LE:32:default`, base address `0x40a000`.
- Full auto-analysis found and decompiled **409 real functions**
  (16 failed to decompile), all located inside the swap region — confirming,
  independently via Ghidra's own call-target-based function discovery, the
  same "swap region is where the real code lives" conclusion the
  branch-consistency sweep found.

### 8.1 Finding the dispatcher

Decompiling all 409 functions and searching for one containing a direct
comparison against `13` or `0xD` (the `ct_special` CSI-enable ID) found
`FUN_0046d7f6` — a generic `(context, id, value)` parameter-setter function
whose case IDs track `ct_special`'s own documented ID list closely (0, 1, 2,
3, 4, 5, 6, 7, 8, 9, 10, 0xb, 0xc, **0xd**, 0xe, 0xf, 0x10, 0x11, ...).

The relevant excerpt:

```c
if (param_2 == 0xc) {
    if (param_3 != 0) {
        (**(code **)(DAT_0042fd34 + -0x1d8))(param_3 & 0xffff);
        return;
    }
}
else if (param_2 == 0xd) {
    if (param_3 != 0) {
        (**(code **)(DAT_0042fd34 + -0x1d4))(param_3 & 0xffff);
        return;
    }
}
else {
    if (param_2 == 0xe) {
        (**(code **)(DAT_0042fd34 + -0x1d0))(param_3 & 0xff);
        return;
    }
    if (param_2 == 0xf) {
        (**(code **)(DAT_0042fd34 + -0x1cc))(param_3 & 0xff);
        return;
    }
    ...
```

Two things make this a strong, specific match rather than a coincidence:

1. **Behavior matches documentation exactly.** `ct_special`'s own help text
   says id `0xD` means *"0 == disable, else enable reporting CSI data."*
   This code does precisely that: calls through a handler only if the value
   is non-zero, otherwise falls through and does nothing.
2. **The call targets form a real function-pointer table.** The offsets for
   ids `0xc, 0xd, 0xe, 0xf` — `-0x1d8, -0x1d4, -0x1d0, -0x1cc` — step by
   exactly 4 bytes each, i.e. consecutive slots in a 32-bit function-pointer
   array, one per feature ID, in ID order. That's the compiled shape of a
   real `switch`/dispatch table, not an artifact of misreading random bytes.

### 8.2 The wall

Resolving what function pointer actually lives at that table slot
(`DAT_0042fd34 - 0x1d4`) hit a dead end: the static value stored there in the
firmware image is not a valid address (`0xaed2a564`, `0xa9b8306` — garbage
as pointers). This strongly suggests the table is **populated at runtime
during firmware boot**, not baked into the static image — a very normal
pattern for embedded "driver ops" tables, but one that static analysis alone
cannot see through. Resolving it would require either tracing the
initialization code that writes each slot, or genuinely running the
firmware (emulation or live hardware with debug access).

## 9. Where this stands / the open question

What's been established with real confidence:

- The exact mechanism (`ct_special` id `0xD`, `WMI_10_4_CSI_MESG_EVENTID`) by
  which this firmware *attempts* to report CSI.
- That on this specific firmware build, under every trigger condition
  tested, the host only ever receives a small, fixed, non-channel-data
  bookkeeping record.
- That this specific firmware's CPU architecture is Xtensa, and its real
  memory layout.
- The exact compiled function and `case` that handles the CSI-enable
  command, and that its behavior matches the documented interface.

What remains **unresolved** — the actual root question this investigation
set out to answer:

> **Does this hardware ever compute and expose real per-subcarrier channel
> data to the host, under any condition, on any firmware revision?**

The internal chip hardware almost certainly *can* compute this (every OFDM
receiver must estimate the channel from the preamble of every packet just to
demodulate it — that's not optional, it's how OFDM works). What's unresolved
is whether Qualcomm/Candela's firmware ever copies that internal
computation across the firmware→host boundary in a form anyone can read —
and static analysis of this one firmware build could not settle that either
way.

## 10. Next steps

The plan going forward (not yet executed as of this writeup): acquire a
second, dedicated EA8300 used purely as an experimentation sandbox, with a
serial (UART) connection to its host SoC's bootloader (U-Boot). This doesn't
give direct debug access to the radio chip's own Xtensa core — it's a
separate processor — but it does provide:

- **Safe recovery.** A bricked OS/kernel can be recovered via U-Boot, where
  a bricked production router over SSH-only access cannot. This is what
  makes more aggressive experiments (custom kernel modules, alternate
  firmware blobs) viable at all.
- **Boot-time visibility** nothing over SSH can show.

The most direct next experiment, once that sandbox exists: port the old
`ath10k-ct` PR #98 relayfs/accumulator patch forward to the current driver
version, to see whether a properly accumulated (rather than naively
per-fragment-dumped) buffer reveals a longer, real payload — since that
patch's own 64-byte minimum length check is *larger* than the 52-byte
buffer this investigation ever observed, raising the possibility that a
genuine, longer CSI report exists under conditions not yet tried.

## 11. Practical methodology notes

A few generalizable things from this session worth keeping, independent of
the ath10k-specific conclusions:

- **`apt-get download <pkg>` + `dpkg-deb -x <pkg>.deb <dir>` needs no root.**
  This was used repeatedly to get working tools (`binutils-xtensa-lx106`,
  `binutils-arc-linux-gnu`, `openjdk-25-jdk-headless` +
  `openjdk-25-jre-headless`) on a machine without sudo access. Watch for
  `.deb` packages whose binaries depend on shared libraries that land in a
  non-default path (`LD_LIBRARY_PATH=...`) or on absolute `/etc` symlinks
  that need to be replaced with real copies when extracted outside a real
  package install.
- **Branch-target self-consistency beats "didn't crash" for unknown-ISA
  identification.** Any heuristic based on "how long before the disassembler
  hits an invalid opcode" is vulnerable to false positives from densely-
  encoded instruction sets. Checking whether branch/call targets land on
  other real instruction boundaries is a much stronger, harder-to-fake
  signal.
- **When call targets fall outside your assumed address range, check whether
  a second segment, loaded contiguously, explains them** before assuming
  the architecture guess is wrong.
- **Always verify a parser against one piece of known ground truth before
  trusting it on the rest of a file.** Here, that was the firmware version
  string, independently knowable from the live router's own debugfs output.
