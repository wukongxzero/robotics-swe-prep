---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [embedded, serial, protocol, struct-packing]

# Serial Packet Protocols

> [!question] Explain it cold
> 
> - What problem does a fixed-size binary packet solve over text/CSV serial?
> - Why does struct padding matter, and how do you get a known wire size?
> - How do you frame/sync a byte stream so the receiver doesn't desync?

---

## Core idea

For high-rate MCU↔MCU or MCU↔host comms, you send **fixed-size binary packets** mapped to a packed struct rather than parsing text. It's compact, fast to (de)serialize, and deterministic — but you have to control struct layout and framing precisely or the bytes don't line up.

## Key facts & formulas

- **Fixed-size binary** beats text: no parsing, no variable length, predictable timing. Critical at control-loop rates.
- **Struct packing / padding**: compilers align members to their size, inserting padding bytes. A struct with a `char` then a `short` may get a pad byte to align the short. To get a _known_ wire size you either: account for padding explicitly, use `__attribute__((packed))`, or order members largest-first.
- **TankStatus protocol (WALL-E V3)** — 16-byte packet:
    - `driveLeft`, `driveRight` (unsigned char) — motor commands
    - 2 bytes **padding** (explicit, for alignment/known size)
    - `eulerX`, `eulerY` (short, 2 bytes each) — attitude
    - `eulerZ` (char) — attitude
    - Total = `TANKSTATUS_PACKET_LENGTH = 16` bytes.
    - The padding is _deliberate_ — it makes the on-wire size match on both sender and receiver regardless of compiler alignment quirks.
- **Odometry packet**: 22-byte variant, same fixed-layout discipline.
- **Framing/sync**: a raw byte stream needs a way to find packet boundaries — a start/sync byte (or magic header), fixed length, and ideally a checksum/CRC. Without it, one dropped byte desyncs every subsequent packet.
- **Endianness**: sender and receiver must agree (AVR is little-endian) — multi-byte fields (`short`) must be interpreted consistently.


## Interview follow-ups

- **Q:** Why send a packed binary struct instead of comma-separated values?
    - **A:** No parsing cost, fixed deterministic size and timing, compact on the wire — matters at control-loop rates. CSV is for debugging/logging, not the hot path.
- **Q:** Your two MCUs agree on the struct definition but fields come out garbage. Why?
    - **A:** Struct padding/alignment differs, or endianness mismatch, or framing desync. Pin the layout (packed / explicit padding), agree on endianness, add a sync byte + checksum.
- **Q:** One byte gets dropped on the line — what happens without framing?
    - **A:** The receiver desyncs and every subsequent packet is misaligned. A sync header + fixed length lets it re-find boundaries; a checksum lets it reject corrupt frames.


## Links

- Related: [[AVR Register Programming]], [[AVR Peripherals]], [[iceoryx Zero-Copy]], [[Motor Control]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Why fixed-size binary packets over text/CSV on a control-rate serial link? ? No parsing cost, fixed deterministic size and timing, compact on the wire. CSV is for logging, not the hot path.

Why does the TankStatus struct include explicit padding bytes? ? To make the on-wire size a known fixed 16 bytes regardless of compiler alignment, so sender and receiver agree byte-for-byte.

Two MCUs share the struct definition but fields decode as garbage — three suspects? ? Struct padding/alignment mismatch, endianness mismatch, or framing desync from a dropped byte.

What does a sync byte + fixed length + checksum buy you in a byte stream? ? Boundary recovery after a dropped byte (re-sync) and rejection of corrupted frames — prevents permanent desync.