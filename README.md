# NAME

read\_lw\_sources

# DESCRIPTION

This is a Perl script which reads and displays Axia Livewire Channel
Advertisements, created as a tool to help understand and describe
the protocol, for which no documentation has been found.

Given that this has been developed purely from the study of real-world
network traffic, there may be errors or omissions in the way that
messages are parsed and decoded. The protocol implementation is not yet
complete, but is sufficient to be useful and provide a platform for
future development. Feedback or patches are welcome.

When run, the script will subscribe to the Livewire Channel Advertisements
multicast group, decode and display the messages as they are received,
along with debugging information to aid understanding and further
development. Various checks are incorporated to test assumptions made
when deciphering the protocol.

This project is the result of a weekend challenge to interpret the protocol,
driven by the frustation of finding it undocumented and stubbornness -
"how hard can it be" :-)

# EXAMPLE

The following produces a running display of received channel advertisements,
by using the `--list-sources` argument and discarding the test and debug
messages normally sent to stderr:

    $ ./read_lw_sources --list-sources 2>/dev/null
    |-----------|-------------|-------|---------------|-----------------|
    | Node Name |     Node IP | LW Ch | Channel Name  |  Multicast Addr |
    |-----------|-------------|-------|---------------|-----------------|
    | StA-Node1 | 10.1.245.60 |  6031 | PGM1          |  239.192.23.143 |
    | StB-Node1 | 10.1.246.60 |  6032 | PGM1          |  239.192.23.144 |
                               ... [snip] ...

Running the script without any arguments will display low level message
details as they are decoded.

    $ ./read_lw_sources

# CONTRIBUTING

Feedback, further documentation, patches and bug reports are welcome, via
github.

# MESSAGE PROTOCOL

The following is based on the understanding gained from analysing real-world
network traffic. It may be wrong - feedback or corrections are welcome.

## Header

All messages start as follows:

    0x03 0x00 0x02 0x07                         [Start sequence]
    0x?? 0x?? 0x?? 0x??                         [Increments with each message]
    0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00     [Eight NUL padding bytes]

## Phrases

The remainder of the message comprises multiple `phrases`, each with the
following structure:

    [4-byte opcode] [1-byte data type] [n-bytes of data (operand)]

## Opcodes

- ADVT

    The advertisement message type. Type 1 is a full channel advertisement,
    Type 2 is a summary advertisement, Types 3 and 4 are yet to be documented.

- ADVV

    A sequence number, which is incremented each time there is a change in the
    channel configuration for a node.

- ATRN

    The node's display name.

- BSID

    The 'to-source' backfeed multicast address

- FSID

    The 'from-source' multicast address.

- HWID

    Set to the last two octets of the node's IPv4 address.

- INDI

    The number of command `phrases` to follow in this section of the message.

- INIP

    The IPv4 unicast address of the node.

- NUMS

    The number of audio sources being described by the message.

- PSID

    The source's Livewire channel number.

- PSNM

    The source's presentation name.

- PVER

    The protocol version being used for the message. 

- S001, S002...

    An opcode beginning with `S` followed by three digits (e.g. S001) inidicates
    the start of a message section describing a source. The numeric part of the
    opcode indicates the source number. The data payload indicates the size of
    the following message section in bytes.

- SHAB

    Indicates whether a source is shareable (data = 0x01) or not (data = 0x00).
    By default, sources are not shareable, meaning they can only be controlled by
    one console at a time and will appear as 'listen-only' sources if assigned to
    additional consoles.

- TERM

    The start of a message section describing the node ('terminal'). The data
    payload indicates the size of the following section in bytes.

- UDPC

    The port being used, normally 4000.

## Data Types

A number of different data types are encountered, with the size and
interpretation of the subsequent data deduced as follows:

- 0x00

    Single-byte unsigned integer.

- 0x01

    Four-byte word representing either a 32-bit unsigned integer (PSID,
    ADVV or LPID opcodes), four-character text field (TYPE opcode), or
    the four octets of a IPv4 address.

- 0x03

    A variable length text field. The first two bytes of data specify
    the number of characters that follow. Strings are NUL terminated and
    should be padded with NUL characters, but sometimes contain
    garbage after the terminating NUL.

- 0x06

    Two-byte 16-bit unsigned integer.

- 0x07

    Single-byte unsigned integer, as per data type 0x00.

- 0x08

    Two-byte 16-bit unsigned integer, as per data type 0x06.

- 0x09

    8-bytes of data follow. In observed network traffic, these have always been
    eight 0x00 NUL padding characters.

## Message Types

The advertisement message type is indicated by a `ADVT` phrase.

Type 1 messages are full channel advertisements, containing details of
every source available from the transmitting node and a sequence number
which increments every time the node's channel configuration changes.

Type 2 messages are summary advertisements, indicating the number of sources
available from the transmitting node and the current sequence number,
allowing receivers to confirm that they have up-to-date channel details.

Message types 3 and 4 have been observed and appear to contain information
about channel/backfeed ownership and usage. They require further investigation
and analysis to document and decode.

## Example Message - Type 1

    0x03 0x00 0x02 0x07                     [Start of Message - always the same    ]
    0xc7 0xc2 0xa0 0x60                     [Increases with each message from node ]
    0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 [Padding - always 8 NUL bytes          ]
                                            [Message body phrases start here       ]
    NEST 0x00 0x04                          [Unknown meaning                       ]
    PVER 0x08 0x00 0x02                     [Protocol Version 2                    ]
    ADVT 0x07 0x01                          [Advertisement Message type 1          ]
                                            [Section ends                          ] 
    TERM 0x06 0x00 0x54                     [0x54: following section has 84 bytes  ]
    INDI 0x00 0x06                          [0x06: following section has 6 phrases ]
    ADVV 0x01 0x00 0x00 0x00 0x98           [Sequence number, increments on change ]
    HWID 0x08 0xXX 0xXX                     [last two octets of node IP            ]
    INIP 0x01 0xXX 0xXX 0xXX 0xXX           [unicast IP address of node            ]
    UDPC 0x08 0x0f 0xa0                     [0x0fa0: UDP port number 4000          ]
    NUMS 0x08 0x00 0x01                     [0x01: 1 source channel                ]
    ATRN 0x03 0x00 0x20 [0xXX * 32 chars]   [0x20: 32 characters follow. Node name ]
                                            [Section ends                          ]
    S001 0x06 0x00 0x65                     [Channel 001. 0x65: 101 bytes follow   ]
    INDI 0x00 0x0b                          [0x0b: following section has 11 phrases]
    PSID 0x01 0x00 0x00 0x17 0x8f           [0x17 0x8f: Livewire Channel 6031      ]
    SHAB 0x07 0x00                          [0x00: source is not 'shareable'       ]
    FSID 0x01 0xef 0xc0 0x17 0x8f           [Source multicast address              ]
    FAST 0x07 0x02                          [Unknown meaning                       ]
    FASM 0x07 0x01                          [Unknown meaning                       ]
    BSID 0x01 0xef 0xc1 0x17 0x8f           [Backfeed multicast address            ]
    BAST 0x07 0x01                          [Unknown meaning                       ]
    BASM 0x07 0x00                          [Unknown meaning                       ]
    LPID 0x01 0x00 0x00 0x17 0x8f           [0x17 0x8f: Livewire channel 6031      ]
    STPL 0x07 0x00                          [Unknown meaning                       ]
    PSNM 0x03 0x00 0x10 [0xXX * 16 chars]   [0x10: 16 chars follow. Channel name   ]

## Example Message - Type 2

    0x03 0x00 0x02 0x07                     [Start of Message - always the same    ]
    0xc8 0x43 0xe3 0x3d                     [Increases with each message from node ] 
    0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 [Padding - always 8 NUL bytes          ]
                                            [Message body phrases start here       ]
    NEST 0x00 0x03                          [Unknown meaning                       ]
    PVER 0x08 0x00 0x02                     [Protocol Version 2                    ]
    ADVT 0x07 0x02                          [Advertisement Message type 2          ]
    TERM 0x06 0x00 0x2d                     [0x2d: following section has 45 bytes  ]
    INDI 0x00 0x05                          [0x05: following section has 5 phrases ]
    ADVV 0x01 0x00 0x00 0x00 0x22           [Sequence number, increments on change ]
    HWID 0x08 0xXX 0xXX                     [last two octets of node IP            ]
    INIP 0x01 0xXX 0xXX 0xXX 0xXX           [unicast IP address of node            ]
    UDPC 0x08 0x0f 0xa0                     [0x0fa0: UDP port number 4000          ]
    NUMS 0x08 0x00 0x05                     [0x05: 5 source channels               ]

# LICENCE

MIT License

Copyright (C) 2021 NP Broadcast Limited

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
