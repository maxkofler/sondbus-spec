# Sondbus transport layer

This document describes the transport layer of the sondbus suite of protocols.
It is responsible for the reliable exchange of commands over a network layer.

# 1 - Overview

The sondbus transport layer is essentially a framework for reliably reading from and writing to memory of a remote device over a protocol.
This allows for a nice compromise of flexibility of the protocol and minimalism in its implementation.
The aspect of minimalism is especially important in the context of embedded systems, as devices on the network do only need to implement a few basic commands to fully participate in the network without hindering other "bigger" devices ability to communicate in a flexible manner.
The flexibility, on the other side allows participants to extend their ability to communicate information in any way imaginable.

This approach of reading and writing memory essentially decouples the bus logic from the bus communication.
Each party reads from and writes to the memory area in an autonomous way.
This approach closely resembles memory-mapped devices, as commonly found in computer systems, essentially extending a computer's memory map over the network to other computers.

# 2 - General Information

This section lists miscellaneous information about this document and the protocol it specifies.

## 2.1 - Endianness

All multi-byte values in this layer are treated in a little-endian form.
This does not influence the payload sections of commands, as arbitrary data can be packaged there.

## 2.2 - Command packing

This document describes the contents of the `command` section of a transmission.
This means that the start of a command is not part of this document, but is rather documented and described in the [Network Layer](03-Network-Layer.md) document.

# 3 - Commands

The core command set of sondbus (`0x00` - `0x0F`) is a minimal set of instructions that must be supported by every device that implements the sondbus protocol.

- Management commands
  - [3.1: `0x00` - `NOP` - NoOp](#31---nop)
  - [3.2: `0x01` - `SYN` - Sync](#32---sync)
- Short commands
  - [3.3: `0x05` - `BWR` - Broadcast Write](#33---bwr---broadcast-write)
  - [3.4: `0x06` - `PRD` - Physically Addressed Read](#34---prd---physically-addressed-read)
  - [3.5: `0x07` - `PWR` - Physically Addressed Write](#35---pwr---physically-addressed-write)
  - [3.6: `0x08` - `LRD` - Logically Addressed Read](#36---lrd---logically-addressed-read)
  - [3.7: `0x09` - `LWR` - Logically Addressed Write](#36---lwr---logically-addressed-write)

## 3.1 - Nop

The `NOP` command (`0x00`) is, as the name implies, a no-operation.
Its payload is nothing and zero-length.

This command yields no response.

## 3.2 - Sync

The `Sync` command (`0x01`) is used to synchronize slaves and the master.
This need arises from the byte-oriented nature of sondbus, as a newly joined slave or errored slave may loose track of the current conversation.
For it to rejoin the communication, this frame type can be used.

This frame type yields no response, as it is used in a broadcasting manner.
If a slave is not in sync, it will not process any commands that are not this one, and thus also yield no responses, making sure an out-of-sync slave cannot disrupt communication.

A frame with a `SYN` command looks as follows:

|        |  Command   | Magic  | Version |  CRC   |
| :----- | :--------: | :----: | :-----: | :----: |
| Source |   Master   | Master | Master  | Master |
| Bytes  | 1 (`0x10`) |   15   |    1    |   1    |

### 3.2.1 - Sync Magic

The `magic` field of the `Sync` command is fixed as the following sequence of 15 bytes.
These 15 bytes + the CRC at the end of the frame should be unique enough to make this frame distinguishable from all other communication.

```hex
1F 2E 3D 4C 5B 6A 79 88 97 A6 B5 C4 D3 E2 F1
```

## 3.3 - BWR - Broadcast Write

The `Broadcast Write` command (`0x05`) can be used to write to all synchronized slave's memories.
This command yields no response, as the responses would collide.
Due to this fact, the master has no indication of whether the command has succeeded or not.

A `BWR` frame looks as follows:

|        |  Command   | Offset | Length |   Data   |  CRC   |
| :----: | :--------: | :----: | :----: | :------: | :----: |
| Source |   Master   | Master | Master |  Master  | Master |
| Bytes  | 1 (`0x05`) |   1    |   1    | [Length] |   1    |

There is no response to this frame, as it is used in a broadcasting manner. This means that the master has no feedback on whether the transaction succeeded or not.

## 3.4 - PRD - Physically Addressed Read

The `Physically Addressed Read - PRD` command (`0x06`) is used to request data from a slave's memory area.
The addressing scheme uses the slave's physical MAC address.
A slave may only respond to this frame, if it is in sync and its MAC address matches the `address` field of the request exactly.

A `PRD` frame looks as follows:

|        |  Command   | Address | Offset | Length |  CRC   |   Data   |  CRC  |
| :----: | :--------: | :-----: | :----: | :----: | :----: | :------: | :---: |
| Source |   Master   | Master  | Master | Master | Master |  Slave   | Slave |
| Bytes  | 1 (`0x06`) |    6    |   1    |   1    |   1    | [Length] |   1   |

The `Data` and last `CRC` sections are filled by the slave to deliver and confirm the data that has been read.

## 3.5 - PWR - Physically Addressed Write

The `Physically Addressed Write - PWR` command (`0x07`) is used to write data to a slave's memory area.
The addressing scheme uses the slave's physical MAC address.
A slave may only respond to this frame, if it is in sync and its MAC address matches the `address` field of the request exactly.

|        |  Command   | Address | Offset | Length |   Data   |  CRC   |  CRC  |
| :----: | :--------: | :-----: | :----: | :----: | :------: | :----: | :---: |
| Source |   Master   | Master  | Master | Master |  Master  | Master | Slave |
| Bytes  | 1 (`0x07`) |    6    |   1    |   1    | [Length] |   1    |   1   |

## 3.6 - LRD - Logically Addressed Read

The `Logically Addressed Read - LRD` command (`0x08`) is used to request data from a slave's memory area.
The addressing scheme uses the logical address.
A slave may only respond to this frame, if it is in sync and its universe and address exactly match the slave's values.

|        |  Command   | Address | Offset | Length |  CRC   |   Data   |  CRC  |
| :----: | :--------: | :-----: | :----: | :----: | :----: | :------: | :---: |
| Source |   Master   | Master  | Master | Master | Master |  Slave   | Slave |
| Bytes  | 1 (`0x08`) |    1    |   1    |   1    |   1    | [Length] |   1   |

## 3.7 - LWR - Logically Addressed Write

The `Logically Addressed Write Request - LWQ` command (`0x09`) is used to write data to a slave's memory area.
The addressing scheme uses the logical address.
A slave may only respond to this frame, if it is in sync and its universe and address exactly match the slave's values.

|        |  Command   | Address | Offset | Length |   Data   |  CRC   |  CRC  |
| :----: | :--------: | :-----: | :----: | :----: | :------: | :----: | :---: |
| Source |   Master   | Master  | Master | Master |  Master  | Master | Slave |
| Bytes  | 1 (`0x09`) |    1    |   1    |   1    | [Length] |   1    |   1   |
