intermud v2.5
*************

Abstract
========
This documents describes how intermud data is send across the internet in the
protocol specification v2.5.
This specification is derived from Zebedee Intermud (aka Intermud 2) and
intends to be compatible to it, i.e. hosts conforming to both protocols should
be able to exchange data. The aim of v2.5 is to deprecate several historic
ambiguities, define a more consistent (stricter, less implementation-defined)
behaviour, add some optional system services and improve reliability and
remove spoofability of MUDs by introducing hash based message authentication
codes.

Introduction
============

Overview
--------
The intermud protocols define, how (players on) different muds can
communicate with each other. There are several different variants.
In version 2 all muds are peers and directly talking to each other. There
is no central router. Version 2.5 keeps this behaviour but intends to
strengthen the P2P character of the intermud by defining a default
behaviour of learning other peers from one known peer.
The participants of the intermud are intended to be MUDs, not
individual players.

Terminology
-----------
The capitalized key words "MUST", "MUST NOT", "REQUIRED", "SHALL",
"SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP
14, RFC 2119 [TERMS].

MUD
  multi user dungeon
intermud peer
  a participant in the intermud
inetd
  a program encoding and sending and receiving and decoding intermud data
peer address
  IP address of a peer (MUD)
MUD name / peer name
  a name (string) for a peer (MUD)
peer identifier
  a unique combination of MUD name, peer address and peer port


Transport layer
===============
Data between intermud peers is sent as UDP packets (datagrams) over
IP.
Each peer listens on one port and uses one to send data. This kind of
transfer is inherently unreliable, but it's fast and doesn't use up
file descriptors.

Packet length (MTU)
-------------------
A peer **MUST** be able to send and receive datagrams of at least 1024
byte length. The default packet length **SHOULD** be 1024 bytes. If a peer
announces a greater possible length limit, that **SHOULD** be used by other peers
when sending packets to this peer.

Packet format
-------------
All information is encoded and transferred as a string of bytes. The header
names **SHOULD** consist of ASCII characters.
Each packet sent consists of a string as follows:

   M:xxx|V:nnn|F:nnn|header1:body1|headerN:bodyN|DATA:body-data

In other words, a header name, followed by a : and then the data
associated with this header. Each field, consisting of a header/body pair, is
separated by the | character. This means that headers and their body cannot
contain the | character. Peers **SHOULD** check for this in outgoing
packets to avoid decoding errors at the receiving end.

The exception to this is the DATA field. If it is present, it **MUST**
be positioned at the end of the packet. Once a DATA header is
found, everything following it is interpreted as the body of the DATA
field. This means it can contain special characters without error and
it is used to carry the main body or data of all packets.

By convention, predefined system fields will use capital letters for
field headers and custom headers used by specific applications will
use lowercase names to avoid clashes.

A header name **MUST** be unique in a packet.

The fields M (hash based message authentication code), V (version) and F
(flags) **MUST** be in this order at the start of the packet before any other
fields.

Fragmented packets
------------------
If a packet exceeds the maximum packet length, it **MUST** be split
(fragmented) into individual packets small enough.
Each fragment **MUST** start with a fragmentation header describing how the
fragments are to be reassembled at the receiving end.

These fragmentation headers are of the format:

  PKT:mudname:packet-id:packet-number/total-packets|M:xxx|rest-of-packet

In this case, the mudname and packet-id combine to form a unique id
for the packet. The packet-number and total-packets information is
used to determine when all buffered packets have been received. The
rest-of-packet part is not parsed, but is stored while the receiver
awaits the other parts of the packet. When/if all parts have been
received they are concatenated and decoded as a normal packet.

Each fragment **MUST** contain its own valid HMAC in the field M.

The sender **SHOULD** send the fragments in the correct order. However, the
receiver **MUST** assume the fragments arrive in any order.

The sender **MUST** send all fragments of a packet within 30 s from sending the
first fragment.
The receiver **MUST** wait for fragments at least 60 s after the first fragment
arrived. After this, the receiver may discard any fragments of this packet and
therefore the packet as a whole.

Packet encoding
---------------
Only 2 generic data types are supported (namely strings and integers). All
other data types **MUST** be expressed as strings or integers.

On encoding integers are simply converted to a corresponding string.
Strings **MUST** be prefixed with the character $. If the first character of a
string is the $ character, it is escaped by prepending another $ character.

Packet decoding
---------------
On decoding, any string with a $ as its first character will have it removed
and will then be treated as a string.
Any other strings will be converted to integers.

The fields M, V and F **SHOULD** be stripped from the packet data that is
transferred from the inetd implementation to the application.

Legacy mode packets and encoding
--------------------------------
Any intermud v2.5 peer **MUST** send data as described above. However, when
receiving it **MUST** accept data in a relaxed format that is sent by older
intermud peers. In legacy mode, the following changes are accepted:

* The M, V and F fields are missing (aka: **MUST NOT** be present) or are not the
  first three header fields.
* A string **MAY** be prefixed with the character $, but does not have to, unless
  there ambiguity as to wether they should be decoded as a string or an
  integer. If a string is losslessly convertable to an integer and back to a
  string, it **MUST** be prefixed by $.
  This means however, that any string not starting with $ **MUST** be checked
  whether it is to be interpreted as integer or string.

If a peer sends to a peer with a known protocol version older than v2.5 it
**MAY** send the data in the legacy mode. However, this is not recommended.


Defined system headers / fields
===============================
The fields defined in this section **MUST NOT** be used in any application sending
data via intermud. The sending inetd **SHOULD** check for this during input
validation before assembling a packet.

"RCPNT" (RECIPIENT)
    The body of this field should contiain the recipient the message
    is to be sent to if applicable.
"REQ" (REQUEST)
    The name of the intermud request that is being made of the
    receiving mud. Standard requests that should be supported by
    all systems are "ping" (PING), "query" (QUERY), and "reply"
    (REPLY). The PING request is used to determine wether or not a
    mud is active. The QUERY request is used to query a remote mud
    for information about itself (look at the udp/query module for
    details of what information can be requested). The REPLY request
    is special in that it is the request name used for all replies
    made to by mud B to an initial request made by a mud A. It is
    mud A's responsibility to keep track of the original request
    type so that the reply can be handled appropriately. 
"SND" (SENDER)
    The name of the person or object which sent the request or to
    whom replies should be directed. This is essential if a reply
    is expected. 
"DATA" (DATA)
    This field should contain the main body of any packet. It is
    the only field that can contain special delimiting characters
    without error.

The following headers are used internally by the inetd and should
not be used by external objects:

"HST" (HOST)
    The IP address of the host from which a request was received.
    This is set by the receiving mud and is not contained in
    outgoing packets. 
"ID" (ID)
    The packet id. This field is simply an integer which is set by
    the sending inetd. The number is incremented each time a packet
    is sent (zero is never used). This field is only needed if a
    reply is expected. REPLY packets _must_ include the original
    request id. This is _not_ done by the inetd. 
"NAME" (NAME)
    The name of the local mud. Used for security checking and to
    update host list information. 
"PKT" (PACKET)
    A special header reserved for packets which have been split.
    See PACKET PROTOCOL / FORMAT. 
"UDP" (UDP_PORT)
    The UDP port the local mud is receiving on. Used for security
    checking and updating host list information. 
"SYS" (SYSTEM)
    Contains special system flags. The only system flag used at
    present is TIME_OUT. This is included in packets returned due
    to an expected reply timing out to differentiate it from an
    actual reply. 


Intermud requests / modules
===========================

Mandatory requests / modules
----------------------------
The following are standard request types that **MUST** be supported
by all systems:

"ping" (PING)
    This module should return a REPLY packet that contains the
    original requests ID in it's ID field and the SENDER in it's
    RECIPIENT field. It should also include an appropriate string
    in the DATA field, eg. "Mud-Name is alive.\n" 
"query" (QUERY)
    This module expects the type of query requested to appear in the
    recieved DATA field. It should return a REPLY packet containing
    the original ID in the ID field, the SENDER in it's RECIPIENT
    field, and the query type in a QUERY field. The DATA field should
    contain the information requested. 


Optional requests / modules
----------------------------

