---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [embedded, serial, protocol, struct-packing]

# Serial Packet Protocols


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


## Links

- Related: [[AVR Register Programming]], [[AVR Peripherals]], [[iceoryx Zero-Copy]], [[Motor Control]]
- Parent: [[00 Knowledge Map]]

---

