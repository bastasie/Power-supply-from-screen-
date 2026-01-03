PPU framebuffer + PowerBridge I/O (real measurements only)
====================================================

This webapp runs a tiny bytecode VM (PPU) loaded from a GIF “cartridge”.
It ingests *real* PV/touch readings through either:
  1) WebSerial (desktop Chrome / supported contexts), or
  2) WebSocket (works on mobile), or
  3) Manual paste (copy/paste the real line protocol).

Nothing in the UI claims the browser is physically powered by the screen.

Files
-----
- ppu_gif_powerbridge_webapp_fixed.html  (single-file webapp)
- demo_cartridge.ppug.gif                (valid demo cartridge)

Cartridge format (PPUG)
----------------------
Each GIF frame is a 16×16 grayscale tile. The red channel (0..255) is read
row-major and yields 256 bytes per frame. Multiple frames = multiple pages.

Header (first 16 bytes in page 0):
  0..3   "PPUG"
  4      version = 1
  5      page_count
  6      tile_w = 16
  7      tile_h = 16
  8      screen_w = 64
  9      screen_h = 64
  10..11 entry_offset (uint16 little-endian, byte address into concatenated pages)
  12     flags (reserved)
  13..15 reserved

VM opcodes
----------
 0x00 NOP
 0x10 LDI   r, imm8
 0x11 ADDI  r, imm8
 0x12 MOV   rd, rs
 0x13 ADD   rd, rs
 0x14 INC   r
 0x15 CMPI  r, imm8   (sets Z)
 0x16 JNZ   addr16
 0x18 JMP   addr16
 0x19 ANDI  r, imm8
 0x20 CLS   imm8
 0x21 PSET  rx, ry, rc
 0x22 FRAME
 0x23 HALT
 0x30 IN    r, port8  (io_in[port] → register r)
 0x31 OUT   port8, r  (register r → io_out[port], forwarded to link)

I/O ports (8 bytes in each direction)
-------------------------------------
IN ports (measurements → VM):
 0..1  Vpv_mV  (uint16 LE)
 2..3  Ipv_uA  (uint16 LE)
 4..5  Ebuf_mJ (uint16 LE; scale on the firmware side if needed)
 6     Ct      (uint8 coarse)
 7     dCt     (int8 encoded as uint8; interpret signed)

OUT ports (VM → hardware link):
 0..7  user-defined. Default mapping used by the UI description:
   - O0..O1 : Vset_mV (uint16 LE)
   - O2..O3 : Iset_uA (uint16 LE)
   - O4     : mux/mode

Line protocol (Serial or WebSocket)
-----------------------------------
Hardware sends newline-delimited key-values:
  Vpv_mV=1234 Ipv_uA=560 Ebuf_mJ=9000 Ct=812 dCt=-3

The webapp sends OUT writes as:
  O0=12 O1=34 O2=...

Notes
-----
- If WebSerial is not supported in your browser (common on mobile), use WebSocket
  by running a tiny bridge on a laptop/phone/hardware that exposes ws://... and
  forwards the same text lines.
