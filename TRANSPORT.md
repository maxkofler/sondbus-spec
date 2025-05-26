# Sondbus transport layer

The sondbus transport layer is essentially a framework for reliably reading from and writing to memory of a remote device over a protocol.
This allows for a nice compromise of flexibility of the protocol and minimalism in its implementation.
The aspect of minimalism is especially important in the context of embedded systems, as devices on the network do only need to implement a few basic commands to fully participate in the network without hindering other "bigger" devices ability to communicate in a flexible manner.
The flexibility, on the other side allows participants to extend their ability to communicate information in any way imaginable.

This approach of reading and writing memory essentially decouples the bus logic from the bus communication.
Each party reads from and writes to the memory area in an autonomous way.
This approach closely resembles memory-mapped devices, as commonly found in computer systems, essentially extending a computer's memory map over the network to other computers.

All multi-byte values are encoded in little-endian fashion.

The transport layer of sondbus comes in two flavors:

- [1: `Single-Command`](#1---single-command): Single commands are cycled through the bus
- [2: `Multi-Command`](#2---multi-command): Multiple commands are cycled through the bus (optional)

Sondbus requires the [Single-Command](#1---single-command) mode to be implemented by all devices that are on the bus. \
The [Multi-Command](#2---multi-command) mode can only be used if **all** devices on the network support this mode.

# 1 - Single-Command

The single-command mode allows only one command in flight at all times.
This mode is well-suited for the smaller embedded side, as the devices on the bus only need space and implementations that cover one single command.

| Start Byte | Command | Payload | CRC |
| :--------: | :-----: | :-----: | :-: |
|     8      |    8    |    n    |  8  |

- `Start Byte`: This byte is always `0x55`
- `Command`: The command to be transmitted
- `Payload`: The optional payload of the command
- `CRC`: A CRC for the frame

# 2 - Multi-Command

The multi-command mode allows a connection to carry multiple commands to be executed in one bus cycle.
This mode is suited to bigger systems that have more processing and memory capacities.

| Start Byte | Number of Commands | Commands | CRC |
| :--------: | :----------------: | :------: | :-: |
|      8     |         16         | [8 + n]  |  8  |

- `Start Byte`: This byte is always `0xAA`
- `Number of Commands`: The number of commands that are wrapped in this frame
- `Commands`: 0 or more commands appended to each other
- `CRC`: A CRC for the frame

# 3 - Commands

- [3.1: `0x00` - `NOP` - NoOp](#31---nop)
- [3.1: `0x10` - `SYN` - Sync](#32---sync)
- [3.2: `0x12` - `BWQ` - Broadcast Write](#33---bwq---broadcast-write)
- [3.3: `0x14` - `PRQ` - Physically Addressed Read Request](#34---prq---physically-addressed-read-request)
- [3.4: `0x16` - `PWQ` - Physically Addressed Write Request](#35---pwq---physically-addressed-write-request)
- [3.5: `0x18` - `XRS` - Read Response](#36---xrs---read-response)
- [3.6: `0x1A` - `XWS` - Write Response](#37---xws---write-response)
- [3.7: `0x2_` - `LRQ` - Logically Addressed Read Request](#38---lrq---logically-addressed-read-request)
- [3.8: `0x4_` - `LWQ` - Logically Addressed Write Request](#39---lwq---logically-addressed-write-request)

## 3.1 - Nop

The `NOP` command (`0x00`) is, as the name implies, a no-operation.
Its payload is nothing and zero-length.

This command yields no response.

## 3.2 - Sync

The `Sync` command (`0x10`) is used to synchronize slaves and the master.
This need arises from the byte-oriented nature of sondbus, as a newly joined slave or errored slave may loose track of the current conversation.
For it to rejoin the communication, this frame type can be used.

This frame type yields no response, as it is used in a broadcasting manner.
If a slave is not in sync, it will not process any commands that are not this one, and thus also yield no responses, making sure an out-of-sync slave cannot disrupt communication.

The `payload` field of the frame contains the following 16 byte structure:

```rust
struct CmdSYN {
  /// A fixed magic of 15 bytes - See `3.2.1 - Sync Magic`
  magic: [u8; 15],

  /// The version of the sondbus protocol the master wants to initiate
  /// - `0`: Reserved  - should **never** be used
  /// - `1`: Version 1.0.0
  version: u8
}
```

### 3.2.1 - Sync Magic

The `magic` field of the `Sync` command is fixed as the following sequence of 15 bytes.
These 15 bytes + the CRC at the end of the frame should be unique enough to make this frame distinguishable from all other communication.

```hex
1F 2E 3D 4C 5B 6A 79 88 97 A6 B5 C4 D3 E2 F1
```

## 3.3 - BWQ - Broadcast Write

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

## 3.4 - PRQ - Physically Addressed Read Request

The `Physically Addressed Read Request - PRQ` command (`0x14`) is used to request data from a slave's memory area.
The addressing scheme uses the slave's physical MAC address.
A slave may only respond to this frame, if it is in sync and its MAC address matches the `address` field of the request exactly.

To deliver the requested data, a slave must use the [`Read Response - XRS`](#36---xrs---read-response) command.

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

## 3.5 - PWQ - Physically Addressed Write Request

The `Physically Addressed Write Request - PWQ` command (`0x16`) is used to write data to a slave's memory area.
The addressing scheme uses the slave's physical MAC address.
A slave may only respond to this frame, if it is in sync and its MAC address matches the `address` field of the request exactly.

To deliver a response over the request, a slave must use the [`Write Response - XWS`](#37---xws---write-response) command.

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

## 3.6 - XRS - Read Response

The `Read Response - XRS` command (`0x18`) is used by slaves to deliver the requested data back to the master.
It is the consequence of one of the following requests:

- [`0x14` - `PRQ` - Physically Addressed Read Request](#34---prq---physically-addressed-read-request)
- [`0x2_` - `LRQ` - Logically Addressed Read Request](#38---lrq---logically-addressed-read-request)

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

## 3.7 - XWS - Write Response

The `Write Response - XWS` command (`0x1A`) is used by slaves to inform the master about the outcome over a write request.
It is the consequence of one of the following requests:

- [`0x16` - `PWQ` - Physically Addressed Write Request](#35---pwq---physically-addressed-write-request)
- [`0x4_` - `LWQ` - Logically Addressed Write Request](#39---lwq---logically-addressed-write-request)

The address is implied, as the master already knows the address of the slave it requested a write from.

This command may only be emitted if a slave is in sync.

```rust
struct CmdXWS {
  /// The result code of the request
  result: u8,
}
```

## 3.8 - LRQ - Logically Addressed Read Request

The `Logically Addressed Read Request - LRQ` command (`0x20` - `0x2F`) is used to request data from a slave's memory area.
The addressing scheme uses the logical address, where the lower nibble of the command is the universe and the slave's address is in the `address` field.
A slave may only respond to this frame, if it is in sync and its universe and address exactly match the slave's values.

To deliver the requested data, a slave must use the [`Read Response - XRS`](#36---xrs---read-response) command.

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

## 3.9 - LWQ - Logically Addressed Write Request

The `Logically Addressed Write Request - LWQ` command (`0x40` - `0x4F`) is used to write data to a slave's memory area.
The addressing scheme uses the logical address, where the lower nibble of the command is the universe and the slave's address is in the `address` field.
A slave may only respond to this frame, if it is in sync and its universe and address exactly match the slave's values.

To deliver a response over the request, a slave must use the [`Write Response - XWS`](#37---xws---write-response) command.

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
