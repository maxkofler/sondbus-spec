# The sondbus protocol version 1.0.0

This document describes the `sondbus` protocol that aims to create a reliable communication channel for embedded devices.

# 0 - About this document

## 0.1 - Structure Layout

Payload structures are explained using syntax of the `Rust` programming language, as its type hints are pretty clear.
This means that the following:

```rust
struct SomeStructure {
    first: u8,
    second: u8,
    third: u16,
}
```

translates to this sequence of bytes:

| first | second | third (h) | third (l) |
| :---: | :----: | :-------: | :-------: |
|  1    |   1    |     1     |     1     |

# 1 - General Information

Sondbus - at its core - is a protocol that facilitates a master reading from and writing to memory of one or more slaves.
This manifests itself in each slave presenting a area of memory to the master that can be configured and accessed by it.

## 1.1 - Data Layout

Sondbus works on a byte-basis, meaning that it does not require padding.

Multi-byte values are encoded in `little-endian` style.

## 1.2 - Layers

Sondbus is composed of 3 main layers:

- [2 - Physical Layer](#2---physical-layer)
- [3 - Data Link Layer](#3---data-link-layer)
- [4 - Application Layer](#4---application-layer)

# 2 - Physical Layer

Sondbus is designed to work over anything that can transmit and receive individual bytes.
This makes the barrier for entry quite low, as no facilities for framed packet handling are required from the environment.
Additionally, it lowers the overhead on the bus, making the communication channel wider for user data.

# 3 - Data Link Layer

The data link layer facilitates transporting read and write instructions from a master to one or multiple slaves and their responses back to the master.
These read and write instructions form the basis for communicating with the slaves.

The data link layer works in a framed manner.
The following structure shows the basic structure of a sondbus frame:

```rust
struct Frame {
  /// The start byte is always 0x55 and indicates the start
  /// of a sondbus frame
  start_byte: u8,

  /// The command byte indicates the command that is carried
  /// as the payload - see `3.1 - Commands`
  command: u8,

  /// The payload that is transmitted in this frame,
  /// depends on the `command` byte
  payload: [u8; ...],

  /// A `CRC 8 Autosar` to check the contents of the frame, includes the following:
  /// - `start_byte`
  /// - `command`
  /// - `payload`
  crc: u8,
}
```

# 3.1 - Commands

- [3.1.1: `0x00` - `SYN` - Sync](#311---sync)
- [3.1.2: `0x12` - `BWQ` - Broadcast Write](#312---bwq---broadcast-write)
- [3.1.3: `0x14` - `PRQ` - Physically Addressed Read Request](#313---prq---physically-addressed-read-request)
- [3.1.4: `0x16` - `PWQ` - Physically Addressed Write Request](#314---pwq---physically-addressed-write-request)
- [3.1.5: `0x18` - `XRS` - Read Response](#315---xrs---read-response)
- [3.1.6: `0x1A` - `XWS` - Write Response](#316---xws---write-response)
- [3.1.7: `0x2_` - `LRQ` - Logically Addressed Read Request](#317---lrq---logically-addressed-read-request)
- [3.1.8: `0x4_` - `LWQ` - Logically Addressed Write Request](#318---lwq---logically-addressed-write-request)

## 3.1.1 - Sync

The `Sync` command (`0x00`) is used to synchronize slaves and the master.
This need arises from the byte-oriented nature of sondbus, as a newly joined slave or errored slave may loose track of the current conversation.
For it to rejoin the communication, this frame type can be used.

This frame type yields no response, as it is used in a broadcasting manner.
If a slave is not in sync, it will not process any commands that are not this one, and thus also yield no responses, making sure an out-of-sync slave cannot disrupt communication.

The `payload` field of the frame contains the following 16 byte structure:

```rust
struct CmdSYN {
  /// A fixed magic of 15 bytes - See `3.1.1.1 - Sync Magic`
  magic: [u8; 15],

  /// The version of the sondbus protocol the master wants to initiate
  /// - `0`: Reserved  - should **never** be used
  /// - `1`: Version 1.0.0
  version: u8
}
```

### 3.1.1.1 - Sync Magic

The `magic` field of the `Sync` command is fixed as the following sequence of 15 bytes.
These 15 bytes + the CRC at the end of the frame should be unique enough to make this frame distinguishable from all other communication.

```hex
1F 2E 3D 4C 5B 6A 79 88 97 A6 B5 C4 D3 E2 F1
```

## 3.1.2 - BWQ - Broadcast Write

The `Broadcast Write` command (`0x12`) can be used to write to all synchronized slave's memories.
This command yields no response, as the responses would collide.
Due to this fact, the master has no indication of whether the command has succeeded or not.

```rust
struct CmdBWQ {
  /// The offset in the slave's memory to write to
  offset: u16,

  /// The length of the following `data` field
  length: u16,

  /// The data to be written to the slave's memory
  data: [u8; length]
}
```

## 3.1.3 - PRQ - Physically Addressed Read Request

The `Physically Addressed Read Request - PRQ` command (`0x14`) is used to request data from a slave's memory area.
The addressing scheme uses the slave's physical MAC address.
A slave may only respond to this frame, if it is in sync and its MAC address matches the `address` field of the request exactly.

To deliver the requested data, a slave must use the [`Read Response - XRS`](#315---xrs---read-response) command.

```rust
struct CmdPRQ {
  /// The address of the slave to read from
  address: [u8; 6],

  /// The offset in the slave's memory to read from
  offset: u16,

  /// The length of the data to be read
  length: u16,
}
```

## 3.1.4 - PWQ - Physically Addressed Write Request

The `Physically Addressed Write Request - PWQ` command (`0x16`) is used to write data to a slave's memory area.
The addressing scheme uses the slave's physical MAC address.
A slave may only respond to this frame, if it is in sync and its MAC address matches the `address` field of the request exactly.

To deliver a response over the request, a slave must use the [`Write Response - XWS`](#316---xws---write-response) command.

```rust
struct CmdPWQ {
  /// The address of the slave to write to
  address: [u8; 6],

  /// The offset in the slave's memory to write to
  offset: u16,

  /// The length of the data to be written
  length: u16,

  /// The data to be written to the slave's memory
  data: [u8; length],
}
```

## 3.1.5 - XRS - Read Response

The `Read Response - XRS` command (`0x18`) is used by slaves to deliver the requested data back to the master.
It is the consequence of one of the following requests:

- [`0x14` - `PRQ` - Physically Addressed Read Request](#313---prq---physically-addressed-read-request)
- [`0x2_` - `LRQ` - Logically Addressed Read Request](#317---lrq---logically-addressed-read-request)

The address and data length are implied, as the master already knows the size of the data requested and the address of the slave it requested this data from.

This command may only be emitted if a slave is in sync.

```rust
struct CmdXRS {
  /// The result code of the request
  result: u8,

  /// The data read from the slave's memory, if successful
  data: [u8; length]
}
```

## 3.1.6 - XWS - Write Response

The `Write Response - XWS` command (`0x1A`) is used by slaves to inform the master about the outcome over a write request.
It is the consequence of one of the following requests:

- [`0x16` - `PWQ` - Physically Addressed Write Request](#314---pwq---physically-addressed-write-request)
- [`0x4_` - `LWQ` - Logically Addressed Write Request](#318---lwq---logically-addressed-write-request)

The address is implied, as the master already knows the address of the slave it requested a write from.

This command may only be emitted if a slave is in sync.

```rust
struct CmdXWS {
  /// The result code of the request
  result: u8,
}
```

## 3.1.7 - LRQ - Logically Addressed Read Request

The `Logically Addressed Read Request - LRQ` command (`0x20` - `0x2F`) is used to request data from a slave's memory area.
The addressing scheme uses the logical address, where the lower nibble of the command is the universe and the slave's address is in the `address` field.
A slave may only respond to this frame, if it is in sync and its universe and address exactly match the slave's values.

To deliver the requested data, a slave must use the [`Read Response - XRS`](#315---xrs---read-response) command.

```rust
struct CmdPRQ {
  /// The address of the slave to read from
  address: u8,

  /// The offset in the slave's memory to read from
  offset: u16,

  /// The length of the data to be read
  length: u16,
}
```

## 3.1.8 - LWQ - Logically Addressed Write Request

The `Logically Addressed Write Request - LWQ` command (`0x40` - `0x4F`) is used to write data to a slave's memory area.
The addressing scheme uses the logical address, where the lower nibble of the command is the universe and the slave's address is in the `address` field.
A slave may only respond to this frame, if it is in sync and its universe and address exactly match the slave's values.

To deliver a response over the request, a slave must use the [`Write Response - XWS`](#316---xws---write-response) command.

```rust
struct CmdLWQ {
  /// The address of the slave to write to
  address: u8,

  /// The offset in the slave's memory to write to
  offset: u16,

  /// The length of the data to be written
  length: u16,

  /// The data to be written to the slave's memory
  data: [u8; length],
}
```

# 4 - Application Layer

The application layer stands above the data link layer in that it represents the various services that are controlled through the memory area and deliver their responses through that.

# 5 - Addressing

Sondbus uses 2 layers of addressing:

- [Physical Addressing](#51---physical-addressing)
- [Logical Addressing](#52---logical-addressing)

## 5.1 - Physical Addressing

The physical address of a slave is fixed in that it is bound to the physical interface that is attached to the bus.
It consists of 48 bits (6 octets) that are equivalent to a MAC address.
This address is hardcoded into each slave and should not change, as data could be bound to it.

## 5.2 - Logical Addressing

A logical address is split into 2 parts:

- Universe (4 bits)
- Address (8 bits)

A universe is a collection of 255 devices, while an address is unique within its universe.

This separation allows for partitioning of the network to group up slaves in a way that allows the master to optimize transfers and broadcasts.

# 6 - Slave synchronization

Sondbus requires a slave to keep exact track of all submitted bytes on the bus.
This requires some mechanism for a slave to synchronize with the bus and 'clock in' to the conversation.

To facilitate this mechanism, all slaves have an internal flag, here called `in_sync` that determines if a slave is synchronized with the bus.
This flag is required for a slave to process **any** data received on the bus.
If it is not set, the slave will only accept one frame: [Sync](#311---sync).
In this out-of-sync state, the slave ignores all bytes, except they line up as the [Sync](#311---sync) frame.

If a slave is out-of-sync (having the `in_sync` flag not asserted), it **must** return to a safe state that does not require bus communication.
