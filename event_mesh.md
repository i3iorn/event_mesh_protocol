# Event Mesh Protocol (EMP)

# Abstract
The Event Mesh Protocol (EMP) is a lightweight, event-driven communication protocol designed to facilitate seamless interaction between distributed systems and services. EMP enables the efficient exchange of events, allowing applications to respond to changes in real-time and promoting a decoupled architecture.
EMP is built to take advantage of the speed of UDP for low-latency event delivery, while also providing mechanisms for increased, but not absolute, reliability and message integrity.

EMP make no guarantees about message delivery, ordering, or duplication. It is the responsibility of the application to handle these aspects if required.
EMP Does not offer encryption or authentication. If required, QUIC can be used as a transport protocol to provide these features.

## Table of Contents
1. **[Overview](# Overview)**
2. **[EMM Packet Structure](# EMM Packet Structure)**
   - 2.1 **[Header Format](# Header Format)**
   -  2.2 **[Payload](# Payload)**
   - 2.3 **[Time to Live (TTL)](# Time to Live (TTL))**
   - 2.4 **[Example EMM Packet](# Example EMM Packet)**
3. **[Protocol Operations](# Protocol Operations)**
   - 3.1 **[NODE Joining](# NODE Joining)**
   - 3.2 **[NODE relationships](# NODE relationships)**
   - 3.3 **[Event Publishing](# Event Publishing)**
   - 3.4 **[Event Receiving and Propagation](# Event Receiving and Propagation)**
   - 3.5 **[Reliability](# Reliability)**
   - 3.6 **[NODE Leaving](# NODE Leaving)**
4. **[Network features](# Network features)**
   - 4.1 **[DSCP usage](# DSCP usage)**
   - 4.2 **[ECN usage](# ECN usage)**
   - 4.3 **[QUIC as a transport protocol](# QUIC as a transport protocol)**
5. **[Security](# Security)**
   - 5.1 **[Replay attacks](# Replay attacks)**
   - 5.2 **[Integrity](# Integrity)**
   - 5.3 **[Confidentiality](# Confidentiality)**
   - 5.4 **[Authentication](# Authentication)**
6. **[Extensions (TLV)](# Extensions (TLV))**
   - 6.1 **[TLV Types](# TLV Types)**

# 1. Overview
The Event Mesh Protocol (EMP) is designed to address the challenges of modern distributed systems, where services need to communicate asynchronously and respond to events in real-time. EMP provides a standardized way to publish events and have them propagate quickly throughout a network of NODEs.

It is a stateless protocol with the goal to quickly and robustly deliver events to interested parties. EMP is particularly well-suited for applications that require high throughput and low latency, such as IoT systems, microservices architectures, and real-time analytics platforms. It can also be used as the basis for an event bus or message broker. However, in such cases some of the functionality must be added by the application.

EMP NODEs can act as both event producers and consumers, allowing for a flexible and dynamic event-driven architecture. The protocol supports various event types, including simple notifications, complex data payloads, and command messages.

NODEs can communicate directly with each other or through intermediary NODEs that can route events based on predefined rules or subscriptions. This allows for a scalable and resilient event mesh that can adapt to changing network conditions and service availability.

NODEs communicate using EMM (Event Mesh Message) packets, which are designed to be compact and efficient. Each EMM packet contains a header with metadata about the event, such as its type, source, and timestamp, as well as the event payload itself. All messages MUST be smaller than the maximum transmission unit (MTU) of the underlying network to avoid fragmentation. 

# 2. EMM Packet Structure
An EMM packet consists of a fixed-size header followed by a variable-size payload. The header contains essential information for routing and processing the event. The payload is variable, but MUST be less than the MTU of the underlying network.
RECOMMENDED maximum size is 512 bytes.

All multi-byte fields are in network byte order (big-endian).

## 2.1 Header Format
The EMM header is 8 bytes long and consists of the following fields:

| Field          | Size | Description                     |
|----------------|------|---------------------------------|
| Version        | 3b   | Protocol version (currently 1)  |
| Flags          | 2b   | Control flags (See 2.1.2)       |
| Type           | 3b   | Event type (See 2.1.1)          |
| Event ID       | 2B   | Unique identifier for the event |
| Epoch          | 2B   |                                 |
| Offset         | 2B   |                                 |
| Payload Length | 1B   | Length of the payload in bytes  |
| Payload        | Var  | Event data (variable length)    |

```text
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Ver.| Fl| Type| Event ID (16 bits)            |Epoch (16 bits)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               | Offset (16 bits)              | Payload Len   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
~                         Payload (variable)                    ~
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

* **Version**: A 3-bit field indicating the version of the EMP protocol. The current version is 1 (0b001).
  * RECOMMENDED default is 0b001
  * Bitmask: 0b11100000
* **Flags**: A 3-bit field used for control flags. See section 2.1.2 for details.
  * RECOMMENDED default is 0b00 (no flags set)
  * Bitmask: 0b00011000
* **Type**: A 2-bit field indicating the type of event. See section 2.1.1 for details.
  * RECOMMENDED default is 0b000 (Notification)
  * Bitmask: 0b00000111
* **Event ID**: A 2-byte field that serves as a unique identifier for the event. This ID is used to prevent duplicate processing of events. 
  * RECOMMENDED to be a random or pseudo-random value to minimize the risk of collisions.
  * The combination of Event ID, Epoch, and Offset MUST be unique for each event generated by a NODE.
  * The Event ID is primarily used to identify and track events as they propagate through the mesh.
  * The Event ID is NOT required to be globally unique across all NODEs, but it MUST be unique for each event generated by a NODE within the context of the Epoch and Offset.
* **Epoch**: A 2-byte modulo 65536 counter that increments every second. This helps to avoid Event ID collisions over time.
  * RECOMMENDED to start at 0 and increment by 1 every minute, wrapping around to 0 after reaching 65535.
  * The Epoch is used in conjunction with the Event ID to provide a larger space for unique event identification.
* **Offset**: A 2-byte field that is used to divide the Epoch into smaller time intervals. This allows for finer granularity in event identification within the same Epoch.
  * RECOMMENDED to start at 0 and increment by 1 for each event generated within the same Epoch, wrapping around to 0 after reaching 65535.
* **Payload Length**: A 1-byte field indicating the length of the payload in bytes. The maximum payload length is 256 bytes.
  * RECOMMENDED default is 0x00 (no payload)
* **Payload**: A variable-length field containing the actual event data. The length of this field is specified by the Payload Length field.

### 2.1.1 Event Types
The Type field can be thought of as the "message type" or "event type". It indicates what kind of event is being transmitted. This helps NODEs to understand how to process the event.
The Type field in the header can take on the following values:

| Value | Name         | Description                                                               |
|-------|--------------|---------------------------------------------------------------------------|
| 0b000 | Notification | A simple notification event, typically used for status updates or alerts. |
| 0b001 | Command      | An event that represents a command to be executed by the receiving NODE.  |
| 0b010 | Heartbeat    | A periodic event used to indicate that a NODE is alive and functioning.   |
| 0b011 | Join         | Used to join a network.                                                   |
| 0b100 | Reserved     | Reserved for future use.                                                  |
| 0b101 | Reserved     | Reserved for future use.                                                  |
| 0b110 | Reserved     | Reserved for future use.                                                  |
| 0b111 | Reserved     | Reserved for future use.                                                  |

* **Notification**: Used for sending simple notifications, such as status updates or alerts. The payload MAY contain a message string or a structured data format (e.g., JSON).
* **Command**: Represents a command that the receiving NODE MAY execute. The payload typically contains the command name and any necessary parameters. There is no guarantee that the command will be executed,
* **Heartbeat**: A periodic event sent by NODEs to indicate that they are alive and functioning. The payload MUST contain the NODE's identifier and MAY include additional status information.
* **Join**: Used by a NODE to join the event mesh. The payload MAY include information about the NODE's capabilities and configuration.
* **Reserved**: These values are reserved for future use and MUST NOT be used in the current version of the protocol.

### 2.1.2 Flags
The Flags field is a bitmask that provides additional information about the event. The following flags are defined:

| Value | Name         | Description                                                             |
|-------|--------------|-------------------------------------------------------------------------|
| 0b00  | Urgent       | Indicates that the event is urgent and should be processed immediately. |
| 0b01  | Reserved     | Reserved for future use.                                                |
| 0b10  | Reserved     | Reserved for future use.                                                |
| 0b11  | Reserved     | Reserved for future use.                                                |

* **Urgent**: When set, this flag indicates that the event is urgent and should be processed immediately by the receiving NODE. This MAY influence the NODE's prioritization of event processing.
* **Reserved**: These bits are reserved for future use and MUST be set to zero.

## 2.2 Payload
The payload contains the actual event data. The format of the payload is determined by the event type and can vary widely. It is the responsibility of the application to define and interpret the payload format based on the event type.

All payloads MUST begin with mandatory TLV for the signature as in section 6. The rest of the payload is application-specific and MUST be defined by the application.

Not all payloads require the full signature. Below is a table showing what level of signature is required for each event type.

| Event Type   | Signature Required |
|--------------|--------------------|
| Notification | 32B                |
| Command      | 64B                |
| Heartbeat    | 16B                |
| Join         | 64B                |

**Validation Logic**
- If the payload does not contain the required signature TLV, the event MUST be discarded.
- If the signature is present but invalid, the event MUST be discarded.
- If the signature is valid, the event MAY be processed further based on its type and payload.
- If the length of the signature does not match the expected length for the event type, the event MUST be discarded.
- If the Type field in the TLV is not 0xFE, the event MUST be discarded.

**Rationale**
The signature is used to verify the authenticity and integrity of the event. Different event types have different security requirements, hence the varying signature lengths. For example, Command events may require a longer signature to ensure that only authorized NODEs can issue commands.

### 2.2.1 Notification Payload

| Field    | Size     | Description                                       |
|----------|----------|---------------------------------------------------|
| Priority | 1B       | Type of notification (e.g., info, warning, error) |
| Format   | 1B       | Format of the message (e.g., plain text, JSON)    |
| Message  | Variable | The actual notification message                   |

For Notification events, the payload MAY contain a message string or a structured data format (e.g., JSON). The format of the payload is application-specific and MUST be defined by the application.
* **Priority**: Indicates the type of notification, such as informational, warning, or error.
  * 0x11 - Trace
  * 0x12 - Debug
  * 0x13 - Info
  * 0x14 - Notice
  * 0x15 - Warning
  * 0x16 - Error
  * 0x17 - Critical
  * 0x18-0x1F - Reserved
* **Format**: Specifies the format of the message, such as plain text or JSON.
  * 0x21 - Plain text
  * 0x22 - JSON
  * 0x23 - XML
  * 0x24 - Binary
  * 0x25 - YAML
  * 0x26 - Protobuf
  * 0x27 - MsgPack
  * 0x28 - CBOR
  * 0x29 - Avro
  * 0x2A - Thrift
  * 0x2B-0x2F - Reserved
* **Message**: The actual notification message, which can be of variable length. The length of this field is determined by the Payload Length field in the header minus the size of the Priority and Format fields (2 bytes).

### 2.2.2 Command Payload

For Command events, the payload typically contains the command name and any necessary parameters. The format of the payload is 
application-specific and MUST be defined by the application. The purpose of the Command event is to request the receiving NODE to perform a specific action. There is no guarantee that the 
command will be executed, as it depends on the capabilities and state of the receiving NODE. An example can be that a NODE sends a command to other nodes to increase logging level or to start/stop a specific service.

A NODE MUST validate the command before executing it. If the command is not recognized or cannot be executed, the NODE MUST ignore the command. If the command is validated the NODE SHOULD execute it.

The command payload MUST include the following TLVs as defined in section 6:
* **Command Name**: A string representing the name of the command to be executed.
* **Command Parameters**: A structured data format (e.g., JSON) containing any parameters required for the command.
* **Signature - Strong(0xA4)**: An Ed25519 signature for the Command event.

See section 6 for details on TLV format and types.

### 2.2.3 Heartbeat Payload
For Heartbeat events, the payload MUST contain the NODE's identifier (e.g., a UUID) and MAY include additional status information, such as CPU usage, memory usage, or uptime. The format of the payload is application-specific.

The heartbeat payload MUST include the following TLVs as defined in section 6:
* **Node ID**: A unique identifier for the NODE, such as a UUID.
* **Uptime**: The NODE's uptime in seconds.
* **Event Digest**: A rolling hash, Bloom filter, or digest of recent events sent or received by the NODE.
* **Rate Signal**: Information about the NODE's event sending and receiving rates, as well as any congestion hints (e.g., ECN status).
* **Propagation Map**: Information about the NODE's position in the mesh, such as hop count from origin, remaining TTL, or origin NODE ID.
* **Signature - Weak(0xA2)**: An Ed25519 signature for the Heartbeat event.

See section 6 for details on TLV format and types.

NODEs MAY include any combination of the above TLVs in the Heartbeat payload. The order of the TLVs is not significant, and NODEs MUST be able to parse the payload regardless of the order of the TLVs.

### 2.2.4 Join Payload
For Join events, the payload MUST include information about the NODE's capabilities and configuration. This information helps other NODEs understand how to interact with the joining NODE.

The join payload MUST include the following TLVs as defined in section 6:
* **Node ID**: A unique identifier for the NODE, such as a UUID.
* **Supported Types**: A bit field indicating the event types supported by the NODE.
* **Max Relationships**: The maximum number of relationships the NODE can support.
* **Heartbeat Interval**: The recommended interval for sending Heartbeat events.
* **Status**: Application-specific status information.
* **Signature - Strong(0xA4)**: An Ed25519 signature for the Join event.

See section 6 for details on TLV format and types.

## 2.4 Time to Live (TTL)
When creating a new event, a NODE MUST set the TTL field in the IP header. This value indicates the maximum number of hops the event can take before being discarded. As any router following the IP protocol decrements the TTL field by 1, a NODE MUST also decrement the TTL field by 1 before propagating the event to its relationship NODEs. If the TTL reaches 0, the event MUST be discarded and not propagated further.

The TTL field is not part of the EMM packet itself but is included in the IP header of the UDP packet carrying the EMM packet. The initial value of the TTL field is application-specific and MAY be based on factors such as network size, expected event propagation distance, and desired event lifetime. RECOMMENDED default is 64.

## 2.5 Example EMM Packet
Here is an example of an EMM packet in hexadecimal format:
```
01 00 00 01 00 01 00 0F 25 FE 20 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 13 21 1F 48 65 6C 6C 6F 2C 20 45 4D 50 21 20 54 68 69 73 20 69 73 20 61 20 74 65 73 74 2E 20 3A 29

# Explained
Header (8 Bytes):
Version: 0b001 (3 bits)
Flags: 0b00 (2 bits, Urgent not set)
Type: 0b000 (3 bits, Notification)
Event ID: 0x0001 (2 Bytes)
Epoch: 0x0001 (2 Bytes)
Offset: 0x000F (2 Bytes)
Payload Length: 0x25 (1 Byte, 37 Bytes payload)
Payload (37 Bytes):
Signature TLV: Type 0xFE (1 Byte), Length 0x20 (1 Byte), Value: 32 Bytes (Ed25519 signature, placeholder)
Priority: 0x13 (1 Byte, Info)
Format: 0x21 (1 Byte, Plain text)
Message Length: 0x1F (1 Byte, 31 Bytes message)
Message: "Hello, EMP! This is a test. :)" (31 Bytes)
```
This packet represents a Notification event (Type 0b000) with the Urgent flag not set. The Event ID is 1, and the timestamp indicates when the event was created. The payload length is 15 bytes, and the payload contains a simple notification message. The checksum is calculated over the entire packet to ensure data integrity.
In this example, the payload might represent a notification message with a priority of "Info" and a format of "Plain text", followed by the actual message content.
The actual interpretation of the payload would depend on the application-specific format defined for Notification events.

# 3. Protocol Operations
## 3.1 NODE Joining
When a NODE wants to join the event mesh, it sends a Join event to announce its presence and capabilities to other NODEs. This event includes information about the supported event types and version information.
Any receiving NODE that supports the Join event type can process this event and update its list of known NODEs in the mesh. Each NODE MUST store enough information about new NODEs to enable them to re-establish a lost connection.

## 3.2 NODE relationships
When a NODE receives a Join event from another NODE, it MAY choose to establish a relationship with that NODE. 
- A NODE MUST accept AT LEAST 10 relationships. Relationships are used to determine which NODEs to send events to and how to route events through the mesh.
- A NODE MAY choose to limit the number of relationships it establishes based on its capabilities and resources.
- A NODE MAY choose to prioritize relationships based on factors such as latency, reliability, or event types supported.
- A NODE MAY choose to drop relationships that are no longer needed or that are not performing well.
- A NODE MAY choose to periodically send Heartbeat events to refresh its relationships.
- A NODE MAY choose to send events to all known NODEs or only to a subset of NODEs based on its routing rules or subscriptions.

## 3.3 Event Publishing
To publish an event, a NODE constructs an EMM packet with the appropriate header and payload, then sends it to its relationship NODEs using UDP. The NODE MAY choose to send the event to all its relationship NODEs or only to a subset based on its routing rules.

## 3.4 Event Receiving and Propagation
When a NODE receives an EMM packet, it first verifies the checksum to ensure data integrity. If the checksum is valid, the NODE processes the event based on its type and payload. The NODE MAY also choose to propagate the event to other NODEs in the mesh, depending on its routing rules or subscriptions. The only reliability mechanism in EMP is that a NODE MUST send each event to all its relationship NODEs.

When a NODE receives an event, it MUST check the Event ID to determine if it has already processed the event. If the Event ID is new, the NODE processes the event and MUST propagate it to its relationship NODEs. If the Event ID has already been seen, the NODE MUST discard the event to prevent duplicate processing.

- A NODEs MUST decrement the TTL field by 1 before propagating the event. If the TTL reaches 0, the event MUST be discarded and not propagated further. 
- A NODE MUST NOT modify any other fields in the EMM packet, except for the TTL and Checksum fields. The Checksum MUST be recalculated after modifying the TTL field.
- A NODE MUST cache Event IDs for a configurable period to prevent processing duplicate events. The cache duration is application-specific and MAY be based on factors such as event frequency and network conditions.
  - RECOMMENDED default cache duration is 5 minutes.
- A NODE MAY implement additional filtering or processing logic based on the event type, flags, or payload content. This logic is application-specific and MUST be defined by the application. This MAY include parsing TLVs as defined in Section 6.

## 3.5 Reliability
EMP is designed to be lightweight and fast, but it does not guarantee absolute reliability. To improve reliability, NODEs can implement application-level acknowledgments or retries for critical events. However, this is optional and depends on the specific use case and requirements of the application.

## 3.6 NODE Leaving
When a NODE wants to leave the event mesh, it can simply stop sending and receiving events. There is no formal leave process in EMP. Other NODEs MAY choose to remove the leaving NODE from their list of known NODEs after a certain timeout period if they do not receive any Heartbeat events from that NODE.

# 4. Network features
## 4.1 DSCP usage
EMP NODEs SHOULD set the DSCP field in the IP header to a value that reflects the priority of the event being transmitted. For example, urgent events MAY be marked with a higher DSCP value to ensure they receive preferential treatment by routers and switches in the network.

| Type  | Type Field = Value  | DSCP Value | Description                               |
|-------|---------------------|------------|-------------------------------------------|
| 0b000 | Priority = Trace    | 0x08       | Low priority, non-critical notifications  |
| 0b000 | Priority = Debug    | 0x10       | Low priority, debugging information       |
| 0b000 | Priority = Info     | 0x18       | Normal priority, informational messages   |
| 0b000 | Priority = Notice   | 0x20       | Normal priority, important notifications  |
| 0b000 | Priority = Warning  | 0x28       | High priority, warning messages           |
| 0b000 | Priority = Error    | 0x30       | High priority, error messages             |
| 0b000 | Priority = Critical | 0x38       | Highest priority, critical alerts         |
| 0b001 | Urgent Flag Set     | 0x38       | Highest priority, urgent commands         |
| 0b001 | Urgent Flag Clear   | 0x20       | Normal priority, non-urgent commands      |
| 0b010 | Any                 | 0x10       | Low priority, periodic heartbeat messages |
| 0b011 | Any                 | 0x20       | Normal priority, join events              |

### 4.1.1 Reference:
- 0x08 - CS1 (Low priority, non-critical traffic)
- 0x10 - CS2 (Low priority, debugging traffic)
- 0x18 - CS3 (Normal priority, general traffic)
- 0x20 - CS4 (Normal priority, important traffic)
- 0x28 - CS5 (High priority, warning traffic)
- 0x30 - CS6 (High priority, error traffic)
- 0x38 - CS7 (Highest priority, critical traffic)
- 0x00 - Best Effort (Default for non-EMP traffic)

## 4.2 ECN usage
EMP NODEs SHOULD set the ECN field in the IP header to indicate their congestion status. The ECN field can take on the following values:
- 0b00 (Not-ECT): The NODE is not using ECN.
- 0b01 (ECT(1)): The NODE is using ECN and is not experiencing congestion.
- 0b10 (ECT(0)): The NODE is using ECN and is not experiencing congestion.
- 0b11 (CE): The NODE is experiencing congestion.

EMP NODEs SHOULD monitor network conditions and adjust their ECN status accordingly. For example, if a NODE detects packet loss or increased latency, it MAY set the ECN field to CE to indicate congestion. Conversely, if the NODE's network conditions improve, it MAY revert the ECN field to ECT(0) or ECT(1).

When a NODE receives an event with the ECN field set to CE, it SHOULD take appropriate action to reduce its transmission rate or implement congestion control mechanisms. This helps prevent further congestion in the network and ensures that events can be delivered reliably.

## 4.3 QUIC as a transport protocol
While EMP is designed to work over UDP, it can also be used over QUIC for applications that require additional features such as encryption, authentication, and improved reliability. When using QUIC as the transport protocol, the EMM packet structure and protocol operations remain the same, but the underlying transport layer provides additional capabilities.
Using QUIC as the transport protocol can be particularly beneficial for applications that require secure communication or need to operate in environments with high packet loss or variable latency. QUIC's built-in congestion control and retransmission mechanisms can help improve the reliability of event delivery in such scenarios.
When using QUIC, NODEs MUST ensure that the EMM packets are properly encapsulated within QUIC frames and that the QUIC connection is established before sending or receiving events. The application MUST handle the QUIC connection management, including connection establishment, termination, and error handling.

# 5. Security
## 5.1 Replay attacks
EMM packets include an Event ID, Epoch, and Offset to help prevent replay attacks. NODEs MUST cache recently seen Event IDs along with their associated Epoch and Offset values for a configurable period (RECOMMENDED default is 5 minutes). If a NODE receives an event with an Event ID, Epoch, and Offset that it has already seen within the cache duration, it MUST discard the event to prevent replay attacks.

## 5.2 Integrity
As EMM packets are atomic and self-contained, the integrity of the packet is ensured by the checksum field in the UDP header. NODEs MUST verify the checksum of received packets before processing them. If the checksum is invalid, the NODE MUST discard the packet.

## 5.3 Confidentiality
EMP does not provide built-in encryption or confidentiality mechanisms. If confidentiality is required, NODEs SHOULD use QUIC as the transport protocol, which provides encryption and secure communication.
Alternatively, NODEs can implement application-level encryption for the payload of EMM packets. In such cases, the application MUST handle key management, encryption, and decryption.

## 5.4 Authentication
EMP does not provide built-in authentication mechanisms. If authentication is required, NODEs SHOULD use QUIC as the transport protocol, which provides authentication through TLS.
Alternatively, NODEs can implement application-level authentication mechanisms, such as digital signatures or HMACs, for the payload of EMM packets. In such cases, the application MUST handle key management and verification.

# 6. Extensions (TLV)
The EMP protocol allows for application-specific extensions using a Type-Length-Value (TLV) format in the payload of EMM packets. This allows applications to include additional information that is not covered by the standard EMM packet structure.

The TLV format consists of the following fields:
| Field  | Size | Description                             |
|--------|------|-----------------------------------------|
| Type   | 6b   | A 6-bit field indicating the type of the TLV |
| Length | 6b   | A 6-bit field indicating the length of the Value field in bytes |
| Value  | Var  | A variable-length field containing the actual value |

## 6.1 TLV Types
The following TLV types are defined for use in EMP:

| Type | Name                  | Length   | Purpose                                                                       |
|------|-----------------------|----------|-------------------------------------------------------------------------------|
| 0xA1 | Payload Encoding      | 4b       | zstd, raw, deflate, etc.                                                      |
| 0xA2 | Signature - Weak      | 4b       | Message integrity                                                             |
| 0xA3 | Signature - Moderate  | 8b       | Higher trust requirement                                                      |
| 0xA4 | Signature - Strong    | 16b      | Highest trust requirement                                                     |
| 0xA5 | Compression algorithm | 4b       | zstd, deflate, lz4, etc.                                                      |
| 0xA6 | Compression level     | 5b       | Optional decompression guidance                                               |
| 0xA7 | Event Priority        | 3b       | Low, normal, high                                                             |
| 0xA8 | Node Fingerprint      | 4B       | Boot nonce, MAC hash, or pubkey hash                                          |
| 0xA9 | Node ID               | 16B      | Unique node identifier                                                        |
| 0xAA | Replay Nonce - Short  | 2B       | Random per-event entropy                                                      |
| 0xAB | Replay Nonce - Long   | 8B       | Higher entropy for stronger anti-replay                                       |
| 0xAC | Expiry Hint           | 1B       | TTL or offset from timestamp                                                  |
| 0xAD | Topology Hint         | variable | Known peers, hop distance, region                                             |
| 0xAE | Trace ID              | 16B      | For observability and correlation                                             |
| 0xAF | Vendor ID             | 16B      | Identifies the implementer                                                    |
| 0xB0 | Supported event types | 1B       | Bitfield of supported event types                                             |
| 0xB1 | Max Relationships     | 6b       | Max number of relationships supported                                         |
| 0xB2 | Heartbeat Interval    | 1B       | Recommended heartbeat interval (in seconds)                                   |
| 0xB3 | Uptime                | 4B       | NODE uptime in seconds                                                        |
| 0xB4 | Event Digest          | 16B      | Rolling hash, Bloom filter, or digest of recent events                        |
| 0xB5 | Rate Signal           | 6B       | Events sent in last interval, events received, ECN status or congestion hints |
| 0xB6 | Propagation Map       | 6B       | Hop count from origin, remaining TTL, origin node ID                          |
| 0xB7 | Command Name          | variable | Name of the command to be executed                                            |
| 0xB8 | Command Parameters    | variable | Parameters required for the command                                           |

* **0xA1 - Payload Encoding**: Indicates the encoding used for the payload (e.g., ASCII, UTF-8). This helps the receiving NODE to correctly decode the payload.
  * Length: 4 bytes
  * Possible values:
  * 0b0001 - Raw (binary)
  * 0b0010 - ASCII
  * 0b0011 - UTF-8
  * 0b0100 - UTF-16
  * 0b0101 - ISO-8859-1
  * 0b0110 - cp1252
  * 0b0111-0b1111 - Reserved
* **0xA2 - Signature - Weak**: A weak signature for message integrity.
  * Length: 4 bytes
  * Use case: Non-critical messages where performance is prioritized over security.
* **0xA3 - Signature - Moderate**: A moderate-strength signature for higher trust requirements.
  * Length: 8 bytes
  * Use case: Important messages that require a balance between security and performance.
* **0xA4 - Signature - Strong**: A strong signature for the highest trust requirements.
  * Length: 16 bytes
  * Use case: Critical messages that require maximum security.
* **0xA5 - Compression Algorithm**: Indicates the compression algorithm used for the payload (e.g., zstd, deflate, lz4).
  * Length: 4 bytes
  * Possible values:
  * 0b0001 - None
  * 0b0010 - zstd
  * 0b0011 - deflate
  * 0b0100 - lz4
  * 0b0101 - gzip
  * 0b0110-0b1111 - Reserved
* **0xA6 - Compression level**: Provides optional guidance for decompression.
  * Length: 5 bytes
  * Interpreted as an integer value from 0 (no compression) to 31 (maximum compression). Higher values indicate more aggressive compression, which may require more processing power to decompress. Different algorithms may have different interpretations of compression levels.
* **0xA7 - Event Priority**: Indicates the priority of the event (e.g., Trace, Debug...).
  * Length: 3 bytes
  * Possible values:
  * 0b001 - Trace
  * 0b010 - Debug
  * 0b011 - Info
  * 0b100 - Notice
  * 0b101 - Warning
  * 0b110 - Error
  * 0b111 - Critical
* **0xA8 - Node Fingerprint**: A fingerprint of the NODE, such as a boot nonce, MAC hash, or public key hash.
  * Length: 4 bytes
  * Use case: Identifying the NODE uniquely within the mesh.
* **0xA9 - Node ID**: A unique identifier for the NODE, such as a UUID.
  * Length: 16 bytes
  * Use case: Identifying the NODE uniquely within the mesh.
* **0xAA - Replay Nonce - Short**: A short random nonce for per-event entropy to prevent replay attacks.
  * Length: 2 bytes
  * Use case: Adding entropy to events to prevent replay attacks.
* **0xAB - Replay Nonce - Long**: A longer random nonce for higher entropy to prevent replay attacks.
  * Length: 8 bytes
  * Use case: Adding higher entropy to events to prevent replay attacks.
* **0xAC - Expiry Hint**: A hint for the expiry of the event, such as TTL or offset from the timestamp.
  * Length: 1 byte
  * Use case: Indicating how long the event is valid.
* **0xAD - Topology Hint**: Information about the NODE's known peers, hop distance, or region.
  * Length: Variable
  * Use case: Providing context about the NODE's position in the mesh.
  * This TLV can contain multiple sub-TLVs to represent different aspects of the topology hint. For example:
    * Sub-TLV 1: Known Peers
      * Type: 0x01
      * Length: N (number of known peers)
      * Value: List of known peer NODE IDs (each NODE ID is 16 bytes)
    * Sub-TLV 2: Hop Distance
      * Type: 0x02
      * Length: 1 byte
      * Value: Hop distance from the origin NODE (0-255)
    * Sub-TLV 3: Region
      * Type: 0x03
      * Length: N (length of region string)
      * Value: Region identifier (e.g., "us-west", "eu-central")
* **0xAE - Trace ID**: A unique identifier for tracing and correlating events.
  * Length: 16 bytes
  * Use case: Observability and correlation of events across the mesh. Globally unique identifier (e.g., UUID).
* **0xAF - Vendor ID**: An identifier for the implementer of the NODE.
  * Length: 16 bytes
  * Use case: Identifying the vendor or organization that implemented the NODE. Globally unique identifier (e.g., UUID).
* **0xB0 - Supported event types**: A bitfield indicating the event types supported by the NODE.
  * Length: 1 byte
  * Bitfield representation:
  * Bit 0: Notification (0b000)
  * Bit 1: Command (0b001)
  * Bit 2: Heartbeat (0b010)
  * Bit 3: Join (0b011)
  * Bits 4-7: Reserved for future event types
* **0xB1 - Max Relationships**: The maximum number of relationships the NODE can support.
  * Length: 6 bits (can be packed into a byte with 2 bits reserved)
  * Use case: Indicating the NODE's capacity for relationships within the mesh.
* **0xB2 - Heartbeat Interval**: The recommended interval for sending Heartbeat events.
  * Length: 1 byte
  * Use case: Suggesting how frequently the NODE should send Heartbeat events (in seconds).
* **0xB3 - Uptime**: The NODE's uptime in seconds.
  * Length: 4 bytes
  * Use case: Providing information about how long the NODE has been running.
* **0xB4 - Event Digest**: A rolling hash, Bloom filter, or digest of recent events that the NODE has processed.
  * Length: 16 bytes
  * Use case: Helping other NODEs to understand what events have been seen recently.
* **0xB5 - Rate Signal**: Information about the rate of events being sent and received by the NODE.
  * Length: 6 bytes
  * Use case: Providing metrics such as the number of events sent in the last interval, the number of events received, and any ECN status or congestion hints.
  * Bit 1-2: ECN status
    * 0b00 - Not-ECT
    * 0b01 - ECT(1)
    * 0b10 - ECT(0)
    * 0b11 - CE
  * Bit 3-9: Events sent in last interval (0-31)
  * Bit 10-16: Events received in last interval (0-31)
* **0xB6 - Propagation Map**: Information about the propagation of events through the mesh.
  * Length: 6 bytes
  * Use case: Providing metrics such as the hop count from the origin, the remaining TTL, and the origin NODE ID (opaque).
  * Bit 1-5: Hop count from origin (0-31)
  * Bit 6-10: Remaining TTL (0-31)
  * Bit 11-16: Reserved
* **0xB7 - Command Name**: The name of the command to be executed.
  * Length: Variable
  * Use case: Specifying the command that the receiving NODE should execute.
* **0xB8 - Command Parameters**: The parameters required for the command.
  * Length: Variable
  * Use case: Providing additional information needed to execute the command, typically in a structured format (e.g., JSON).