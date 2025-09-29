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

Sondbus defines a set of commands that build the core of the communication infrastructure.
All commands must be implemented by all slaves to be compliant with this specification.
There is the exception of [Optional Features](#4---optional-features) which specified some features that can be available on a per-slave basis.
These features are clearly marked as optional.

| Bit Position |                     Description                     |
| :----------: | :-------------------------------------------------: |
|    0 - 4     | [Command Set Specific](#311---command-set-specific) |
|      5       | [Command Set Selector](#312---command-set-selector) |
|    6 - 7     |      [Sequence Number](#313---sequence-number)      |

### 3.1.1 - Command Set Specific

These bits are specific to the command set that is selected by the [Command Set Selector](#312---command-set-selector).

### 3.1.2 - Command Set Selector

This bit selects between the two available command sets:

- `true` => [Memory Command Set](#32-memory-command-set)
- `false` => [Management Command Set](#33---management-command-set)

### 3.1.3 - Sequence Number

These bits provide sequencing information to the slave.
The sequence number goes from `0` to `3` and rolls over at the end.
If a slave receives a command that is out of the sequence, it will loose sync with the bus.

## 3.2 Memory Command Set

This command set allows access to the memory region defined and exposed by a slave.
The 5 command set specific bits are explained in the following table:

| Bit Position |                        Description                         |
| :----------: | :--------------------------------------------------------: |
|       0      |               [Operation](#3211---operation)               |
|    1 + 2     |   [Slave addressing mode](#3212---slave-addressing-mode)   |
|      3       | [Operation Offset Length](#3213---operation-offset-length) |
|      4       |   [Operation Size Length](#3214---operation-size-length)   |

A frame using the Memory Command Set has the following general structure:

| Length in Octets |     Source     |                    Description                    |
| :--------------: | :------------: | :-----------------------------------------------: |
|        1         |     Master     |                       Start                       |
|        1         |     Master     |             [Command](#321---command)             |
|    0 / 2 / 6     |     Master     |       [Slave Address](#322---slave-address)       |
|      1 / 2       |     Master     |              [Offset](#323---offset)              |
|      1 / 2       |     Master     |                [Size](#324---size)                |
|        1         |     Master     |          [Header CRC](#325---header-crc)          |
|        n         | Master / Slave |             [Payload](#326---payload)             |
|        1         | Master / Slave |                 [CRC](#327---crc)                 |
|        1         |     Slave      | [Acknowledgement CRC](#328---acknowledgement-crc) |

### 3.2.1 - Command

This section explains the 5 command-set specific bits in the `Command` byte:

#### 3.2.1.1 - Operation

This bit indicates whether the master requests a read or write operation:

- `false` => Read
- `true` => Write

#### 3.2.1.2 - Slave addressing mode

These 2 bits indicate the mode by which the targeted slave is addressed:

- `0` => No address, broadcast
- `1` => Physically Addressed by MAC
- `2` => Logically Addressed by Logical Address (optional - See [Optional Features](#4---optional-features))
- `3` => No address, logical memory operation using MMUs (optional - See [Optional Features](#4---optional-features))

#### 3.2.1.3 - Operation Offset Length

This bit indicates the length of the [Offset](#323---offset) field of the command:

- `false` => 8 bit
- `true` => 16 bit (optional)

#### 3.2.1.4 - Operation Size Length

This bit indicates the length of the [Length](#324---length) field of the command:

- `false` => 8 bit
- `true` => 16 bit (optional)

### 3.2.2 - Slave Address

This field contains the address of the slave that this command is targeted at.
It depends in the setting of the [Slave Addressing Mode](#3212---slave-addressing-mode) bits of the [Command](#321---command) byte of the command:

- `0` => 0 bytes
- `1` => 6 bytes for the MAC address of the slave
- `2` => 2 bytes for the logical address of the slave
- `3` => 0 bytes

### 3.2.3 - Offset

The `Offset` field contains the offset at which the operation is performed in the slave's memory area.
The size of this field depends on the [Operation Offset Length](#3213---operation-offset-length) bit in the [Command](#321---command) byte of the command:

- `false` => 8 bit (1 byte)
- `true` => 16 bit (2 bytes)

The 16-bit addressing mode is optional to each slave and can be supported in 2 ways, explained in the [Optional Features](#4---optional-features) section of this document.

### 3.2.4 - Size

The `Size` field contains the amount of octets that are to be transferred by this command.
The size of this field depends on the [Operation Size Length](#3213---operation-size-length) bit in the [Command](#321---command) byte of the command:

- `false` => 8 bit (1 byte)
- `true` => 16 bit (2 bytes)

The 16-bit addressing mode is optional to each slave and can be supported in 2 ways, explained in the [Optional Features](#4---optional-features) section of this document.

### 3.2.5 - Header CRC

The `Header CRC` field contains a CRC of the frame contents up to this point.
It confirms the contents of the header to the slave to inform it on whether it can trust the submitted data or not.

### 3.2.6 - Payload

The `Data` field contains the payload data.
Depending on the `Operation` bit in the `Command`, this data is provided by:

- `false` => Slave
- `true` => Master

### 3.2.7 - CRC

The `CRC` field contains a CRC of the frame contents up to this point.
It confirms to the receiving end the data that has been transmitted.
Depending on the `Operation` bit in the `Command`, this field is sent by:

- `false` => Slave
- `true` => Master

### 3.2.8 - Acknowledgement CRC

This field is an acknowledgement to the write operation by the master.
It only applies to non-broadcast write operations and contains all the fields of the write command including the CRC sent by the master.

## 3.3 - Management Command Set

This set of commands is used to interface with the slaves in a way that does not access the memory region of individual slaves.
There are 32 commands available in this set, of which a subset is reserved for future use and are to be enabled as optional features in the future.

The following commands are specified:

- `0x00`: [NOP](#331---nop)
- `0x01`: [Sync](#332---sync)

### 3.3.1 - NOP

No Operation.

### 3.3.2 - Sync

The synchronization command essentially enables a slave's transceiver.
If a slave first boots up, it will listen for and react only to this command.
This is required, as the lower layer of sondbus may not always provide a framed transmission, so the synchronization has to be handled here.

| Length in Octets | Source |                 Description                  |
| :--------------: | :----: | :------------------------------------------: |
|        1         | Master |                    Start                     |
|        1         | Master |          [Command](#321---command)           |
|        15        | Master |       [Sync Magic](#3321---sync-magic)       |
|        1         | Master | [Protocol Version](#3322---protocol-version) |
|        1         | Master |              [CRC](#327---crc)               |

### 3.3.2.1 - Sync Magic

This field of the `Sync` command is fixed as the following sequence of 15 bytes.
These 15 bytes + the CRC at the end of the frame should be unique enough to make this frame distinguishable from all other communication.

```hex
1F 2E 3D 4C 5B 6A 79 88 97 A6 B5 C4 D3 E2 F1
```

### 3.3.2.2 - Protocol Version

This byte of the `Sync` command is used to indicate to the slaves the feature set that is being used in this communication establishment.
This byte will change rarely (if ever), as it might lock out some devices that do not implement the version.

If a slave does not support the feature set indicated by this version, it may not continue to take part in the communication, as if it never got synchronized.

Currently, there is only one value for this field:

- `0x01`: Version 1.0.0 of the sondbus protocol.

All other values may be treated as invalid and shall fail synchronization.

# 4 - Optional Features

Some features are marked as optional to allow for minimal implementations of this bus system.
A master is in charge to find and use a subset of the available features and to not violate the limits of any device.
A feature can be implemented in 2 levels:

- [Partial Support](#41---partial-support)
- [Full Support](#42---full-support)

If a master does not respect the slave-imposed limits on features, the slave will go out of sync.

## 4.1 - Partial Support

A slave can partially support a feature in the sense that it can tolerate the feature being used on the bus, meaning that it does not get out of sync if the feature is being used on other slaves.
This means that the slave can receive the data enabled by the feature, but it cannot be targeted by it.

An example can explain this best:
If one slave partially supports the 16-bit addressing modes of the [Memory Command Set](#32-memory-command-set) and another supports it fully, it is ok for the master to use the 16-bit addressing modes on the slave that fully supports it, as the partially supporting slave can tolerate and parse them.
The master can, however, never target the partially supporting slave with a 16-bit addressed command, as it cannot handle them - that would require [Full Support](#42---full-support).

## 4.2 - Full Support

A slave fully supports a feature if it can also be targeted by this feature.
