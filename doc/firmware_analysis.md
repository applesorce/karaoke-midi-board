# 03-SC-R5 program ROM analysis

Source: `bin/roland-03-sc-r5-karaoke-MBM29F800BA@TSOP48.BIN` (IC9, Fujitsu MBM29F800BA, 1 MB).
SHA1 `1dbd75511318487d917b5327332719c7c91a0111`. The file is in programmer byte order: every 16-bit word is
byte-swapped relative to the H8/500 bus. `tools/swap16.py` converts it; all addresses below are CPU addresses in the
swapped image (program ROM is mapped at H'000000-H'0FFFFF).

Tools: `tools/h8500dasm/` (MAME's H8/500 disassembler, standalone) — `dasm` linear, `rd` recursive descent from the
vector table.

## Identity

| Item | Value | Where |
|---|---|---|
| CPU code | H8/500 maximum mode (H8/510), reset vector H'008D1A | H'000000 |
| Model strings | `GS-64 VER=3.00` + `SC-88ProVL BOARD` / `03-SC-R` / `SC-88ProVL` / `SC-88Pro` | H'00FD60 |
| Build header | `SC88PRO (MK03B) ` followed by `01 01 00 01 00 10 97 01 16 09 54` (read as 1997-01-16 09:54, BCD) | H'02FF80 |

The firmware is SC-88Pro family code. It is not SC-8850 firmware (the SC-8850 runs an SH-2 with an
`XP-GS` / `SC-GS` program per MAME `roland_sc8850.cpp`). MAME names the VE-GSPro's program `SC88PRO (MK03B)` too.

## Compared with the SC-88Pro's own firmware

Reference: a real SC-88Pro's IC26 (`bin/SC-88PRO-AT27C800@DIP42.BIN`, CRC32 `820824d2` in CPU byte order, MAME's
"1.04"; 88emu's registry lists the same MD5 `9d4c2f12…` as "1.02").

| Item | 03-SC-R5 (this ROM) | SC-88Pro |
|---|---|---|
| Version string | `1.06   1.01` at H'0CF488 | `1.04   1.00` at H'0CF45C |
| Date after the version (BCD) | `00 04 21 21 04` | `97 02 18 12 11` (MAME: 1997-02-18 12:11) |
| Date block at H'00FDE1 | `06 00 04 21 21 02 36` | `03 96 09 16 17 18 25` |
| Build header H'02FF80 | `SC88PRO (MK03B)` 1997-01-16 09:54 | identical |
| Model name table H'00FD60 | `SC-88ProVL BOARD` / `03-SC-R` / `SC-88ProVL` / `SC-88Pro` | identical, same address |
| Wave ROM layout H'02FFB0, per-chip sums | 5 x 4 MB, `2A04 4857 D589 9061 2B18` (at H'0C3BF8) | identical (sums at H'0C3BD6) |

Reading the two date fields the same way, this ROM is a later revision of the same firmware: 1.06, built
2000-04-21 (the year byte `00` taken as 2000), against the SC-88Pro's 1.04 of 1997-02-18. The `03-SC-R` model
name is already in the SC-88Pro's own firmware, so one image serves several boards; how it picks one (port 7 straps
are read into C0:FE38 at reset) is not traced.

Byte comparison, 64 KB page by page:

| Pages | Difference |
|---|---|
| 02-0B, 0E | none: byte-identical. This includes page 02 (layout table, build header) and pages 03-0B |
| 00 | 248 bytes. Almost all are addresses moved by +4 or +H'68 (code shifted in pages 01/0C), plus the H'00FDE1 date block |
| 01 | 891 bytes changed, one 4-byte insertion at H'010E78 (karaoke side), the rest address shifts |
| 0C | 3516 bytes changed, includes the ROM test code (tables shifted by H'22) |
| 0D | 10985 bytes changed, insertions from H'0D84B6 (Test Mode / ROMSET screen code) |
| 0F | the last word only (H'0FFFFE: `91 CD` vs `0C 1F`), the program checksum balance |

Method for the next table: the two images are aligned byte-wise per page; differences that are only moved
addresses (absolute operands, branch displacements and pointer tables shifted by the alignment's own deltas
+4, +7, +8, +H'22, +H'2B, +H'2C, +H'5D, +H'68) are dropped. 122 changed spots remain. Karaoke-side addresses:

| Where | What changed | Confidence |
|---|---|---|
| H'01B017-H'01B069 (new) | A routine that sets DP = H'F0 (the LSP, MB87837, host window in MAME's map) and writes two 5-byte groups to F0:0004..0000: `06 00 00 01 98`, then `C8 50 00 01 A0`, each after BSR H'9717. Called from the init pair at H'01B006 / H'01B00D. Not in the SC-88Pro's 1.04 | code read; meaning of the LSP values unknown |
| H'0C348B | A test-mode dispatcher grew from two choices (R3 = 1, 2) to three (R3 = 1, 2, 3); the new one loops on PJSR @H'01B1D1, a new routine next to the one above | code read |
| H'0DA71F-H'0DCF0A | Table of 16-byte records (`4E CA 03 60`, item number, pointers, value pairs such as `00 7F`, `34 4C`) rebuilt: some groups larger, the last much smaller (SC-88Pro 6733 bytes vs 781). Looks like the front panel / LCD menu definitions | structure only; "menu" is a guess from the shape |
| H'010E78, H'018FC1 | 4 bytes inserted in each | not decoded |
| H'00FDE1, H'0CF488 | version and dates (table above) | read |

Where the tone and drum parameter tables sit is not traced, so whether they changed is not known; the wave ROM
layout and per-chip sums being identical means both expect the same 20 MB of wave data (the SC-88Pro dump in
`build/sc88pro/wave/` matches those sums). What the code changes in pages 01, 0C and 0D do is not analysed.

## Memory map used by the code

Same as MAME `roland_sc88.cpp` `sc88pro_map`:

| Page | Use | Evidence |
|---|---|---|
| H'00-H'0F | program ROM | vectors, PJSR targets |
| H'C0 | battery RAM (DP=C0 loaded 371 times; stack at C0:F800) | reset code |
| H'C8 | XP (RA09-002) host window (DP=C8 loaded 53 times) | XP register accesses |
| H'E0 / H'EF / H'F0 | sub CPU / gate array / LSP | DP loads |

## On-chip peripherals

- **SCI1 / SCI2 are never initialised.** In the whole image the only absolute references to H'FEC8-H'FED5 are
  `BCLR #7,@H'FECA` (H'001704) and `BCLR #7,@H'FED2` (H'00173D): the TXI handlers (vectors H'D8, H'E8) clearing TIE
  and their DTC enable bit. No SMR/BRR/SCR write exists in any addressing form (absolute, @aa:8, register pointer).
  RXI/ERI vectors point at the default handler H'0001F4. Port 8 (TXD/RXD/SCK pins) is never configured.
- Watchdog (H'FF10/11, RSTCSR H'FF1E/1F): never referenced.
- Reset init (H'008D1A): P3DDR=F8, P3DR=00, P4DDR=F4, P4DR=80, P5DDR=00, P6DDR=FF, P6DR=FF, RFSHCR=00 (refresh off),
  IPRA=17, IPRB=55, IPRC=IPRD=00, WCR=00, BRCR=01, ARBT=DF, AR3T=F8. Port 7 (straps) is read into C0:FE38.

## Wave ROM access

The firmware reads the wave ROMs itself in its ROM test (H'0C3B3D, H'0C3C62), with the same sequence the
sc-mcu-wave-romdumper SC-88 dumper uses:

```
MOV.W R2,@H'3922     ; bank: chip select << 4 | 1 MB page
MOV.W R0,@H'3920     ; page: 1 KB page in the 1 MB bank
MOV.W @(H'3C00,R1),R3 ; dummy read of the window word
BTST  #7,@H'391C     ; wait while busy
MOV.W @H'3910,R3     ; the word
SWAP.B R3            ; low byte = earlier ROM byte
```

ROM layout table at H'02FFB0 (8 bytes per chip select): chips 0-4 = flag H'11, 4 x 1 MB; chips 5-7 = H'FF (absent).
That is the SC-88Pro's 5 x 4 MB = 20 MB layout.

Expected per-chip sums at H'0C3BF8, one word per chip select 0-7: `2A04 4857 D589 9061 2B18 FFFF FFFF FFFF`.
H'0C3C62 computes them: for bank register `cs << 4 | 0..3`, page 0-H'3FF, window offset 0-H'3FE, it adds the
SWAP.B'd window word into a 16-bit sum. Since the low byte of the raw word is the earlier ROM byte, that equals the
16-bit sum of the port-order byte stream read as big-endian words. `tools/uart_dump.py parse` checks a dump against
these.

## Flash writes

None. The only AAAA/5555/0555/0AAA constants are XP internal RAM march tests (H'0C3EA0-H'0C4BA0) and a multiply
constant (H'009BCC). The firmware never programs its own flash.

## Program ROM checksum

H'0C3772 sums all words byte-swapped (pages 0F-01 whole, page 00 below H'FE80, plus H'FF40) and expects 0.
In programmer byte order that is the sum of the file's big-endian words. The original image passes (result 0000). It is called from test-mode code (H'0C3186, H'0CCB80); no call from the
normal boot path was found, but coverage of indirect jumps is incomplete, so a patched image is re-balanced
with one compensation word in free space: `tools/fix_checksum.py` writes it at H'0FD5E, the last word of the
trampoline gap (patched image: file bytes `C4 2E`).

## Patch site (sc-mcu-wave-romdumper `cli/patch_88.js`)

| Item | Value |
|---|---|
| Hook (DT1 40 1x 17 handler) | H'06C9B, table entry H'15030 |
| Load area | C0:6104, 908 bytes (bulk dump 49 10 00) |
| Request area | load + H'380 (Drum Map Name) |
| Free gap for the trampoline | H'00E996-H'00FD5F, 5066 bytes of H'FF, unreferenced |
| Trigger | `F0 41 10 42 12 40 10 17 08 00 11 F7` |

Bulk dump regions match the SC-88Pro dumper exactly (user tone banks C0:7A40/7FC0, user drum sets C0:8540/8AD0).

`patch_88.js` applied to this image changes the table entry H'15030 (`6C 9B` -> `E9 96`) and writes the 22-byte
trampoline at H'0E996 (`STC.B DP,@-SP / LDC.B #H'C0,DP / CMP:G.W #H'C0DA,@H'6106 / BNE / PJSR @H'C06104 /
LDC.B @SP+,DP / JMP @H'6C9B`). With the checksum word that is 26 changed bytes in all. The 40 1x 17 handler keeps
R0, R1, R2 and R4 live across the hook; the dumper saves and restores R0-FP, DP, EP and SR.

Reset code sets TP = H'C0 and SP = H'F800, so the dumper's FP-relative variables (TP page) are in RAM page C0.

## UART dumper

| File | What |
|---|---|
| `src/dumper_uart.asm` | RAM resident dumper, loaded at C0:6104, 444 bytes. Request format and frame format in its header |
| `tools/build.sh` | assembles it (AS in `/opt/asl-bld306/bin`), patches the ROM, rebalances the checksum, writes the SysEx |
| `build/fw_patched.bin` | ROM to burn into IC9, programmer byte order, SHA1 `f38a3c12ec5fd6975f16597ef804925cc979d9ce` |
| `build/syx/` | `load.syx` (14 DT1 to 49 10 00-49 16 40, 32 bytes each), `hello.syx`, `dump_cs0.syx`-`dump_cs4.syx` |
| `tools/capture.py` | raw capture from the USB-UART |
| `tools/uart_dump.py parse` | checks each 1 KB frame (s1/s2), writes `wave_csN_port.bin` (4 MB, port order), compares the chip sum |

Default output is SCI1, TXD1 = pin 94, SMR = H'00, BRR = H'00 (`tools/build.sh --sci 1` for TXD2). At 312500 bit/s
one chip is 4 MB + framing, about 135 s; interrupts stay masked for the whole run.

Not verified on hardware yet:

- The bit rate. 312500 bit/s assumes the SCI clock is the 20 MHz crystal / 64 at BRR = 0. If `hello` comes out as
  garbage at 312500, capture again at 156250.
- That TE = 1 alone switches P85 to TXD with port 8 DDR left at its reset value. The firmware never writes P8DDR,
  and the dumper does not either (it is write-only and also sets the IRQ pins' direction).
- The drum map address step (64 bytes per mm step) comes from one data point in sc-mcu-wave-romdumper
  (`49 1E 00` = byte H'380); `load.syx` keeps every message inside one 64-byte block.
- The H8/510 TXD is a 5 V output; the USB-UART input has to accept it.
