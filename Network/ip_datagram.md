## 0x00 About

IP lays on the third layer of ISO model, which is `network layer`

IP address is consist of four one-byte unsigned number, an IP address is a 32-bit number subdivided into four bytes

> IP address ::= {\<net-id>, \<host-id>}

CIDR presentation
> IP address ::= {\<net-prefix>, \<host-id>} (also with slash notation)

ex: 128.14.35.7/20

## 0x01 Datagram Conposition

### header part

Fixed header of `20 bytes`, Flexable of `40 bytes`, which means [20 - 60] bytes range header.

Contains `source-addr` and `destination-addr`

### data part

The MTU(maximum Transfer unit), which means max data section length in mac, is about to be 1500 bytes. So if datagram length exceeded the MTU value size, we need to get datagram fragmented.

## 0x02

ip