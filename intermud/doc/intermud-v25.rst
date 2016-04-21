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
  a unique combination of MUD name, peer address and receiving peer port


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

A peer may announce the largest reliable packet (maximum transmission unit,
maximum size of datagram) it can receive when asked with the QUERY module
which should be the preferred way.

If the MTU cannot be determined with a QUERY, the two peers should try to
determine them by sending heartbeat packets of increasing size to the other
peer (see below).

The packet size that is used for sending **SHOULD** be the smaller of the
maximum packet length of the two communicating peers.

Packet format
-------------
All information is encoded and transferred as a string of bytes. The header
names **SHOULD** consist of ASCII characters.
Each packet sent consists of a string as follows:

   S:xxx|V:nnn|F:nnn|header1:body1|headerN:bodyN|DATA:body-data

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

The fields S (packet signature), V (version) and F (flags) **MUST** be in this
order at the start of the packet before any other fields. This 3 fields are
also referred to as the 'packet header'. The general layout of packets is:

   [fragmentation header]|packet header|packet payload/data

The packet header **MUST NOT** be larger than 512 bytes.

By convention, predefined system fields will use capital letters for
field headers and custom headers used by specific applications will
use lowercase names to avoid clashes.

A header name **MUST** be unique in a packet.


Fragmented packets
------------------
If a packet exceeds the maximum packet length, it **MUST** be split
(fragmented) into individual packets small enough.
Each fragment **MUST** start with a fragmentation header describing how the
fragments are to be reassembled at the receiving end.

These fragmentation headers are of the format:

  PKT:peername:packet-id:packet-number/total-packets|S:xxx|rest-of-packet

In this case, the mudname and packet-id combine to form a unique id
for the packet. The packet-number and total-packets information is
used to determine when all buffered packets have been received. The
rest-of-packet part is not parsed, but is stored while the receiver
awaits the other parts of the packet. When/if all parts have been
received they are concatenated (without the fragmentation header and M fields
of the individual fragments) and decoded as a normal packet.

When storing fragments of a packet, the receiver **MUST** use a unique packet
id which uses the peer name, peer address and sending peer port and the sent
packet-id.

Any peer **MUST** support at least 100 fragments per packet.

Each fragment **MUST** contain its own valid signature in the field S.

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

Packet signatures
-----------------
For packet validation and to prevent tampering on the wire and spoofing of
peers, each packet sent **MUST** contain a field S containing the EC-DSA
signature of the packet.

The first byte of the MAC field specifies the method and curve used. In intermud
v2.5 the following algorithms **MUST** be supported:

*
*
*

The recommended method is ...

The transferred data is the complete packet string **without** the field S.
After the packet (or fragment) is encoded (without the field S), the signature
is calculated using the private EC key and then inserted into the packet
string either at the beginning of the packet or (for fragments) at the end of
the fragmentation header.

Packet validation
-----------------
Upon receiving a fragment or packet, the receiver **MUST** first try to
validate the signature in the field S, if a public key for the sending peer is
known. The receiver extracts the whole field from the received string and
verifies the signature. If signature can't be verified, the receiver **MUST**
discard the fragment or packet.

Fragments are then stored until the packet is completed or the timeout is
exceeded.

The receiver **SHOULD** parse and decode the packet only after this initial
validation. If the packet is malformed and cannot be parsed, the receiver
**MUST** discard the packet.

The intermud protocol versions of peers **SHOULD** be stored and no packets in
an older protocol version **SHOULD** be accepted.

Packet decoding
---------------
On decoding, any string with a $ as its first character will have it removed
and will then be treated as a string.
Any other strings will be converted to integers.

The fields S, V and F **SHOULD** be stripped from the packet data that is
transferred from the inetd implementation to the application.

Legacy mode packets and encoding
--------------------------------
Any intermud v2.5 peer **MUST** send data as described above. However, unless
in a so-called strict mode, a receiving peer **MUST** accept data in a relaxed
format that is sent by older intermud peers. Unless in strict mode, the following
deviations are acceptable when receiving:

* The packet header (S, V and F fields) is missing.
* A string **MAY** be prefixed with the character $, but does not have to, unless
  there ambiguity as to wether they should be decoded as a string or an
  integer. If a string is losslessly convertable to an integer and back to a
  string, it **MUST** be prefixed by $.
  This means however, that any string not starting with $ **MUST** be checked
  whether it is to be interpreted as integer or string.

However, a packet **MUST NOT** be parsed as legacy mode packet, if one of the
following conditions are met:

* the packet contains the field S
* the packet contains a version field F with a version of at least 2500
* the receiving peer operates in strict mode

After a packet conforming to protocol version >= 2.5 (>=2500) was received
from a peer (this implies the succesful validation of the signature), legacy mode
packets from that peer **MUST NOT** be accepted without manual intervention of
an operator or expiration of the peer from the peer list.

If a peer sends to a peer with a known protocol version older than v2.5 it
**MAY** send the data as a legacy mode packet. However, this is not recommended.

Strict mode
-----------
To prevent spoofing of other muds, an operator MAY decide to operate in strict
mode. In this mode, the peer accepts intermud v2.5 packets with a valid S
field only and discards all other packets.
In other words, it disables the compatibility with peers older than v2.5 and
does not communicate with unknown peers.

Request bookkeeping
-------------------
When sending a request that expects/requires an answer, the sender **MUST**
keep track of the request to relate any answers to the original request.

Any peer **MUST** be able to keep track of at least 100 requests.

If the answer of a request does not arrive within 60s, the request **SHOULD**
be expired (timeout).


Host list / Peer data
=====================
A peer **MUST** store the following data about other known peers:

* peer name (unique)
* public key (unique), if available
* peer address
* peer port (receiving)
* time of last contact
* time of first contact
* reputation (trust) of that peer

A peer **SHOULD** store the following data about other known peers:

* list of supported services
* last seen intermud version
* expiration time
* MTU of the peer

A peers public key would be the best unique identifier. However, in the
intermud a peer needs a unique symbolic name to address it. So a peers name
and its public key should both be used as unique and long-lived identifier.

But a peer MAY change its name by either announcing it or by just using a new
name. If the public key remains the same, the entry in the peer list should
be updated accordingly.

If a peer claims to have a name that already exists, but its public key does
not match the known public key of the existing peer entry, the new peer **MUST
NOT** be entered in the peer list. Instead, any packets from that peer
**SHOULD** be discarded. An implementation MAY notify the operator about this.

Reputation
----------
The reputation is a score that symbolizes how trustworthy a peer is. It may be
used for a number of decisions. By default, the reputation score is used as a
scaling factor when exchanging peer information (see below) and it influences
how quickly a peer is expired once it can't be reached.

By default, a new peer starts with a score of 0 (which basically means, the
information it offers, is not trusted). After a peer has been known for some
time, its score gets increased:

==========  ==============
time known  score increase
==========  ==============
7 days      +1
3 months    +1
1 year      +1
==========  ==============

A reputation of more than 3 can only be assigned by an operator.

Peer expiration
---------------
A peer should expire peers from its host list some time after the last contact. The
expiration time may be chosen by the operator.

However, to prevent rogue peers impersonating other peers, peers **MUST NOT**
be expired before 48h or a time this peer announced earlier (see module...
TODO) passed without contact.

==========  ===============
reputation  expiration time
==========  ===============
0           48h
1           7 days
2           3 months
3           6 months
4+          12 months
==========  ===============

If a peer announces it wants to be remembered for longer than 48h without
contact, this wish MAY be respected and the decision MAY be based on its
reputation.

An implementation **MAY** may move offline peers to a separate list for
bookkeeping after some time and stop trying to contact it anymore. This keeps
the active peer list short and efficient. However the 'long offline' peers
should still be remembered to keep the binding of public key and name.

Before expiring a peer, a ping **SHOULD** be sent to check for reachability.

Automatic update of peer data
-----------------------------
When receiving a v2.5 packet with valid HMAC from an address and/or port that
differs from the one in the peer list, the peer entry **SHOULD** be updated to
the new address/port.

If the address or port of a peer changes, this peer **SHOULD** send a ping to
known peers to announce the new address or port.

When receiving a legacy mode packet, the peer entry **MAY** be updated.
However, this carries the risk of rogue peers successfully impersonating
another peer for an extended time.

An inetd **SHOULD** contact the known peers at least once per 24h to check if
it is still online and reachable (ping or helo).

Update of the public key
------------------------
There ist a way to perform an update of the public key without operator
intervention. The new public key **MUST** be received in a v2.5 packet with
valid signature.

A peer may inform other peers about an update of its public key by
  sending a push notification - TODO fill in module - Such an
  update **SHOULD** be honored.


Defined system headers / fields
===============================
The fields defined in this section **MUST NOT** be used in any application sending
data via intermud. The sending inetd **SHOULD** check for this during input
validation before assembling a packet.

RCPNT
    (RECIPIENT) The body of this field should contiain the recipient the message
    is to be sent to if applicable.
REQ
    (REQUEST) The name of the intermud request that is being made of the
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
SND
    (SENDER) The name of the person or object which sent the request or to
    whom replies should be directed. This is essential if a reply
    is expected.
DATA
    This field should contain the main body of any packet. It is
    the only field that can contain special delimiting characters
    without error.

The following headers are used internally by the inetd and should
not be used by external objects:

HST
    (HOST) The IP address of the host from which a request was received.
    This is set by the receiving mud and is not contained in
    outgoing packets.
ID
    The packet id. This field is simply an integer which is set by
    the sending inetd. The number is incremented each time a packet
    is sent (zero is never used). This field is only needed if a
    reply is expected. REPLY packets _must_ include the original
    request id. This is _not_ done by the inetd. 
NAME
    The name of the local mud. Used for security checking and to
    update host list information. 
PKT
    (PACKET) A special header reserved for packets which have been fragmented.
UDP
    The UDP port the local mud is receiving on. Used for security
    checking and updating host list information. 
SYS
    (SYSTEM) Contains special system flags. The only system flag used at
    present is TIME_OUT. This is included in packets returned due
    to an expected reply timing out to differentiate it from an
    actual reply. 


Intermud requests / modules
===========================

Mandatory requests / modules
----------------------------
The following are standard request types that **MUST** be supported
by all systems:

ping
^^^^
This module should return a REPLY packet that contains the
original requests ID in it's ID field and the SENDER in it's
RECIPIENT field. It should also include an appropriate string
in the DATA field, eg. "Mud-Name is alive.\n" 

helo
^^^^
Used to exchange information like the public key.

query
^^^^^
This module expects the type of query requested to appear in the
recieved DATA field. It should return a REPLY packet containing
the original ID in the ID field, the SENDER in it's RECIPIENT
field, and the query type in a QUERY field. The DATA field should
contain the information requested.
TODO: include asking for peer list in JSON format.


Optional requests / modules
----------------------------
These modules are completely optional and their availability at the discretion
of the operator of a peer.


Exchange of secrets for the HMAC
================================
In this draft the secrets should be either exchanged manually between
operators or sent with a push update to known peers.
For the german MUDs participating in the Intermud, the mailing list
mudadmins-de@groups.google.com is available.

