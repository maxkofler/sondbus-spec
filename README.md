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

# 4 - Application Layer

The application layer stands above the data link layer in that it represents the various services that are controlled through the memory area and deliver their responses through that.
