# Event Mesh Protocol (EMP)

# Abstract
The Event Mesh Protocol (EMP) is a lightweight, event-driven communication protocol designed to facilitate seamless interaction between distributed systems and services. EMP enables the efficient exchange of events, allowing applications to respond to changes in real-time and promoting a decoupled architecture.
EMP is built to take advantage of the speed of UDP for low-latency event delivery, while also providing mechanisms for message integrity and authenticity.

EMP makes no guarantees about message delivery, ordering, or duplication. It is the responsibility of the application to handle these aspects if required.
EMP Does not offer secrecy by default. If required, QUIC can be used as a transport protocol to provide these features.


# 1. Overview
The Event Mesh Protocol (EMP) is designed to address the challenges of modern distributed systems, where services need to communicate asynchronously and respond to events in real-time. EMP provides a standardized way to publish events and have them propagate quickly throughout a network of NODEs.

The goal is to quickly and robustly deliver events to interested parties while maintaining as little state as possible. EMP is particularly well-suited for applications that require high throughput and low latency, such as IoT systems, microservices architectures, and real-time analytics platforms. It can also be used as the basis for an event bus or message broker. However, in such cases some of the functionality must be added by the application.

EMP NODEs can act as both event producers and consumers, allowing for a flexible and dynamic event-driven architecture.

NODEs can communicate directly with each other or through intermediary NODEs that can route events based on predefined rules. This allows for a scalable and resilient event mesh that can adapt to changing network conditions and service availability.

NODEs communicate using EMM (Event Mesh Message) packets, which are designed to be compact and efficient. Each EMM packet contains a header with metadata about the event, such as its type, source, and timestamp, as well as the event payload itself. All messages MUST be smaller than the maximum transmission unit (MTU) of the underlying network to avoid fragmentation as the protocol does not handle fragmentation.

> EMP does not handle fragmentation. It is the responsibility of the application to ensure that messages are smaller than the MTU of the underlying network. 
> Recommended limit will try to account for common MTU sizes, but may not be suitable for all networks.

# 2. EMM Packet Structure
An EMM packet consists of a fixed-size header followed by a variable-size payload. The header contains essential information for routing and processing the event. The payload is variable, but MUST be less than the MTU of the underlying network.
RECOMMENDED maximum size is [MAX_PACKET_SIZE](#max_packet_size).

All multi-byte fields are in network byte order (big-endian).

## 2.1 Header Format table

| Field          | Size (bytes)                                           | Description                          |
|----------------|--------------------------------------------------------|--------------------------------------|
| Version        | 1                                                      | Protocol version (currently 1)       |
| Message ID     | 4                                                      | Unique identifier for the message    |
| Flags          | 1                                                      | Bitfield                             |
| Event Type     | 1                                                      | Type of the event                    |
| Timestamp      | 8                                                      | Unix timestamp (seconds since epoch) |
| Payload Length | 2                                                      | Length of the payload in bytes       |
| Payload        | Variable (up to [MAX_PAYLOAD_SIZE](#max_payload_size)) | The event data                       |

# 2.2 Header Field Descriptions
### Version
A field indicating the version of the EMP protocol being used. This allows for future enhancements and backward compatibility.
Currently, the only valid value is 1 (0x01).
### Message ID
A unique identifier for the message, allowing for tracking and correlation of events. This can be used for debugging and logging purposes. Message ID MUST be cryptographically random to avoid collisions.
### Flags
A bitfield providing additional information about the EMM packet. See section 2.3 for more details.
### Event Type
A field indicating the type of event being sent. See section 3 for more details.
### Timestamp
The time the event was created, represented as a Unix timestamp (seconds since epoch). This helps in ordering events and handling time-sensitive data.
### Payload Length
The length of the payload in bytes. This allows the receiver to know how much data to read for the payload.
### Payload
The actual event data being transmitted. The payload is structured as a set of TLV (Type-Length-Value) encoded fields to allow for flexibility and extensibility. The simplest one being a single string or binary blob. See section 4 for more details.

## 2.3 Flags Bitfield
The Flags field is a bitfield that provides additional information about the EMM packet. The following flags are defined:
- **Bit 0 (0x01)**: Urgent - Indicates that the event is urgent and should be processed with higher priority.
- **Bit 1 (0x02)**: Encryption - Indicates that the payload is encrypted. The application layer MUST handle decryption.
  - If this bit is set, the payload MUST include an Encryption Metadata field (Type 0x11) to provide necessary metadata for decryption.
- **Bit 2 (0x04)**: Compressed - Indicates that the payload is compressed. The application layer MUST handle decompression.
- **Bits 3-7**: Reserved for future use. MUST be set to 0 in version 1. Applications MUST ignore these bits if they are set to 1.

# 3. Event Types
Event types are represented as a single byte, allowing for up to 256 different event types. The following event types are defined:
- **0x01**: Hello - Used for NODE discovery and initial handshake.
- **0x02**: Heartbeat - Sent periodically to indicate that a NODE is alive.
- **0x03**: Event - The core event message type used to transmit event data.

# 3.1 Minimal Event Types
Below are the minimal event types and their expected payload structures. Additional information can be included in the payload as needed, but the core fields MUST be present.

## 3.1 Hello EMM
The Hello event is used for NODE discovery and initial handshake. It contains information about the NODE, such as its ID and capabilities. The payload of a Hello event typically includes the following fields:
- NODE ID (Type 0x14)
- Capabilities (Type 0x04) - A binary blob describing the NODE's capabilities.

## 3.2 Heartbeat EMM
The Heartbeat event is sent periodically by a NODE to indicate that it is alive and functioning. The payload of a Heartbeat event typically includes the following fields:
- NODE ID (Type 0x14)
- Timestamp (Type 0x15) - The time the heartbeat was sent.

## 3.3 Event EMM
The Event event is the core message type used to transmit event data. The payload of an Event typically includes the following fields:
- Event Name (Type 0x01) - A string identifying the event.

# 4. Payload Structure
The payload of an EMM packet is designed to be flexible and extensible. It is structured as a series of TLV (Type-Length-Value) encoded fields. Each field consists of:
- **Type**: A 1-byte identifier indicating the type of the field.
- **Length**: A 1-byte field indicating the length of the Value in bytes.
- **Value**: The actual data of the field, with a length specified by the Length field.

This structure allows for easy addition of new field types without breaking compatibility with existing implementations. NODEs MUST ignore unknown field types, allowing for forward compatibility.

The length of a single field is limited to 255 bytes due to the 1-byte Length field. If a field's data exceeds this limit, it MUST be split into multiple fields of the same Type, each with its own Length and Value.
> Rationale: This keeps the protocol simple and avoids the need for more complex length encoding. And also makes it less likely to accidentally exceed the [MAX_PAYLOAD_SIZE](#max_payload_size).

## 4.1 Core Field Types
The following field types are defined as core types that all NODEs should recognize:
- **0x01**: String - A UTF-8 encoded string.
- **0x02**: Integer - A 4-byte signed integer in network byte order.
- **0x03**: Float - A 4-byte IEEE 754 big-endian floating-point number in network byte order.
- **0x04**: Binary - A binary blob of data.
- **0x05**: JSON - A UTF-8 encoded JSON object.

## 4.2 Security Field Types
To enhance security, the following field types are defined:
- **0x10**: HMAC - A 32-byte HMAC-SHA256 of the canonical bytes for integrity verification.
  - The HMAC is computed over the exact bytes: the 17-byte header as sent (Version, Message ID, Flags, Event Type, Timestamp, Payload Length) concatenated with the canonicalized payload bytes (see Canonicalization below), excluding the HMAC (0x10) and Signature (0x12) fields themselves.
  - The HMAC field MUST appear after all data fields and (if present) after Public Key (0x13) and Auth Key ID (0x18), and MUST appear immediately before Signature (0x12) if a signature is present.
  - If the HMAC field is present, the receiver MUST verify the HMAC in constant-time before processing the payload. If the verification fails, the payload MUST be discarded.

- **0x11**: Encryption Metadata - Metadata required for decrypting the payload, such as IV or key identifiers.
  - This field contains information necessary for decrypting the payload if encryption is used.
  - The actual encryption of the payload is outside the scope of EMP and must be handled by the application layer.
  - If the Encryption Metadata field is present, the receiver MUST use the provided metadata to decrypt the payload before processing it.
  - See [table](#421-encryption-metadata-format) for the format of the Encryption Metadata field.

- **0x12**: Signature - A digital signature of the canonical bytes for authenticity verification.
  - The signature is computed over the exact same canonical bytes as HMAC (header || canonicalized payload excluding 0x10 and 0x12).
  - The Signature field MUST be the last field in the payload if present.
  - If the Signature field is present, the receiver MUST verify the signature using the sender's public key before processing the payload. If the verification fails, the payload MUST be discarded.
  - The public key used for signature verification MUST be provided in a separate Public Key field (Type 0x13) or be retrievable via Auth Key ID (0x18) from a local trust store.
  - The signature algorithm used is Ed25519.
  - The Signature field is 64 bytes in length.
  - 0x12 Signature and 0x10 HMAC fields can coexist in the same payload, allowing for both integrity and authenticity verification. If both fields are present, the receiver MUST verify the HMAC first, followed by the signature.

- **0x13**: Public Key - A public key used for signature verification. Only Ed25519 is supported.
  - This field contains the public key corresponding to the private key used to generate the Signature field (Type 0x12).
  - The Public Key field is 32 bytes in length.
  - If the Signature field is present and the receiver does not have a pre-shared key for the NODE/Key ID, the Public Key field MUST be present.
  - The Public Key field MUST be placed before the Signature field in the payload.
  - The Public Key field is optional if the receiver already has the sender's public key through other means (e.g., pre-shared or obtained via a trusted directory) and can be referenced via Auth Key ID (0x18).

- **0x14**: NODE ID - A unique identifier for the NODE sending the payload.
  - This field contains a unique identifier for the NODE, which can be used for tracking and auditing purposes.
  - The NODE ID is a 16-byte UUID (Universally Unique Identifier).

- **0x15**: Timestamp - A 8-byte Unix timestamp indicating when the payload was signed.
  - This field provides a timestamp for when the payload was signed, which can be used to prevent replay attacks.
  - The timestamp is represented as a Unix timestamp (seconds since epoch).
  - Only required if the Signature field (Type 0x12) is present and the event was signed at a different time than the event creation time.
    - This allows for scenarios where the event is created and then signed at a later time.

- **0x16**: Nonce - A unique nonce used in the encryption process to ensure uniqueness.
  - This field contains a nonce (number used once) that is used in the encryption process to ensure that the same plaintext encrypted multiple times will yield different ciphertexts.
  - The nonce is a 12-byte random value.
  - Only required if the Encryption Metadata field (Type 0x11) is present and the encryption algorithm requires a nonce.

- **0x17**: Key Identifier - An identifier for the encryption key used to encrypt the payload.
  - This field contains an identifier for the encryption key used to encrypt the payload, allowing the receiver to determine which key to use for decryption.
  - The Key Identifier is a 16-byte value.
  - Only required if the Encryption Metadata field (Type 0x11) is present and multiple keys are in use.
  - By default, NODE ID (0x14) can be used as a key identifier if no other key management system is in place.

- **0x18**: Auth Key ID - A short identifier for the key used for HMAC or Signature verification.
  - Exactly 4 bytes. Using a fixed size keeps packets small and simplifies implementations.
  - When present with Signature (0x12), the receiver MUST resolve the Ed25519 public key from a local trust store keyed by (NODE ID, Auth Key ID). When present with HMAC (0x10), the receiver MUST resolve the shared secret from the same mapping.
  - If both Auth Key ID (0x18) and Public Key (0x13) are present, Public Key takes precedence for this packet only.
  - Implementations SHOULD cache the resolved key material per (NODE ID, Auth Key ID) to avoid including Public Key (0x13) in subsequent packets and to keep steady-state packets small.

### Canonicalization (for crypto)
- Canonical bytes = Header (exact 17 bytes as sent) concatenated with Canonical Payload Bytes.
- Canonical Payload Bytes are produced by:
  - Removing HMAC (0x10) and Signature (0x12) fields.
  - Sorting remaining TLVs by Type (ascending). For identical Type values, preserve original relative order.
  - If a TLV contains sub-TLVs (e.g., Encryption Metadata 0x11), canonicalize its sub-TLVs by the same rule.
- The Public Key (0x13), NODE ID (0x14), Timestamp (0x15), Nonce (0x16), Key Identifier (0x17), and Auth Key ID (0x18) are included in the canonical payload if present.

### 4.2.1 Encryption Metadata Format
The Encryption Metadata field (Type 0x11) is structured as a series of TLV encoded sub-fields. The following sub-field types are defined:
- **0x01**: Algorithm - A 1-byte identifier indicating the encryption algorithm used (e.g., 0x01 for AES-256-GCM).
- **0x02**: Length - A 1-byte field indicating the length of the IV in bytes. The most significant bit (MSB) is reserved and MUST be set to 0.
  - This allows for IV lengths up to 127 bytes, which is sufficient for most encryption algorithms.
  - If present, this Length value MUST exactly match the Length of the IV sub-TLV (0x03). Receivers MUST validate this and reject the packet on mismatch.
- **0x03**: IV - A variable-length field containing the initialization vector (IV) used for encryption.
  - The length of the IV depends on the encryption algorithm used (e.g., 12 bytes for AES-256-GCM). For AES-GCM this IV is the nonce.
- **0x04**: Nonce - A 12-byte nonce used in the encryption process to ensure uniqueness.
- **0x05**: Key Identifier - A 16-byte identifier for the encryption key used to encrypt the payload.

The Encryption Metadata field MUST contain at least the Algorithm and either the IV (0x03) or the Nonce (0x04) sub-field, depending on the algorithm. Implementations MUST NOT include both IV and Nonce in the same 0x11 field.

Implementations MUST ensure that all sub-TLV lengths fit within the parent 0x11 TLV Length; sub-TLVs MUST NOT overrun the parent value length.

### 4.2.2 Available Encryption Algorithms
- **0x01**: AES-256-GCM - Advanced Encryption Standard (AES) in Galois/Counter Mode (GCM) with a 256-bit key.
  - Uses a 12-byte IV (nonce). Include IV (0x03); do not include Nonce (0x04).
  - The application layer MUST handle the encryption and decryption process using this algorithm.
- **0x02**: ChaCha20-Poly1305 - ChaCha20 stream cipher with Poly1305 message authentication.
  - Uses a 12-byte nonce. Include Nonce (0x04); do not include IV (0x03).
  - The application layer MUST handle the encryption and decryption process using this algorithm.

## 4.3 Example Payload
An example payload containing a String field, an Integer field, and an HMAC field might look like this (in hexadecimal):

# 5. Security
## 5.1 Integrity
To ensure message integrity, an optional HMAC field (Type 0x10) can be included in the payload. The HMAC is computed using the SHA-256 hash function and a shared secret key known to both the sender and receiver over the canonical bytes (header as sent || canonicalized payload excluding 0x10 and 0x12). If the HMAC field is present, the receiver MUST verify the HMAC in constant-time before processing the payload. If the verification fails, the payload MUST be discarded. When using HMAC, Auth Key ID (0x18) MAY be used to look up the shared secret.

## 5.2 Confidentiality
EMP does not provide built-in encryption for the payload. If confidentiality is required, the application layer MUST handle encryption and decryption of the payload. The Encryption Metadata field (Type 0x11) can be used to provide necessary metadata for decryption, such as IV or key identifiers.

## 5.3 Authenticity
To ensure authenticity, an optional Signature field (Type 0x12) can be included in the payload. The signature is computed using the Ed25519 algorithm over the canonical bytes (header as sent || canonicalized payload excluding 0x10 and 0x12) and the sender's private key. If the Signature field is present, the receiver MUST verify the signature using the sender's public key before processing the payload. If the verification fails, the payload MUST be discarded. The public key used for signature verification MUST be provided in a separate Public Key field (Type 0x13) or referenced via Auth Key ID (0x18) in a local trust store keyed by (NODE ID, Auth Key ID).

## 5.4 Replay Protection
To protect against replay attacks, receivers MUST primarily use the (NODE ID 0x14, Message ID header) pair to detect duplicates within the [REPLAY_WINDOW](#replay_window). Message IDs MUST be cryptographically random; reuse within the window MUST be treated as replay and rejected.

Optionally, the Timestamp field (Type 0x15) can be included to add freshness constraints (e.g., reject messages older than the window or outside [CLOCK_SKEW_ALLOWANCE](#clock_skew_allowance)). When both mechanisms are available, Message ID-based replay detection takes precedence; Timestamp serves as an additional freshness check.

The application layer MAY implement additional logic (e.g., per-event or per-topic de-duplication) if needed.

# 6. Packet creation and parsing
## 6.1 Packet Creation
1. Construct the payload by encoding the desired fields using the TLV format.
   1. If encryption is used, include the Encryption Metadata field (Type 0x11) in the payload.
   2. If HMAC (Type 0x10) or Signature (Type 0x12) is used, then NODE ID (Type 0x14) MUST also be included.
   3. If Signature (Type 0x12) is used and the event was signed at a different time than the event creation time, then Timestamp (Type 0x15) MUST also be included.
   4. If using pre-shared keys or pre-registered public keys, include Auth Key ID (0x18) and omit Public Key (0x13). Include Public Key only when the receiver may not have the key.
2. Order TLV fields as follows:
   - Data fields by 1-byte Type in ascending order. This also applies recursively to sub-fields like in Encryption Metadata (0x11).
   - Security fields last, in this order if present:
     - Public Key (0x13)
     - Auth Key ID (0x18)
     - HMAC (0x10) Covers canonical bytes (header + canonicalized payload excluding 0x10 and 0x12)
     - Signature (0x12) Covers canonical bytes (header + canonicalized payload excluding 0x10 and 0x12); MUST be last
3. Calculate the Payload Length as the total length of the payload.
4. Construct the header with the appropriate values for Version, Message ID, Flags, Event Type, Timestamp, and Payload Length.
5. Concatenate the header and payload to form the complete EMM packet.
6. Send the EMM packet over UDP to the desired destination.

## 6.2 Packet Parsing
1. Receive the EMM packet over UDP.
2. Extract the header fields: Version, Message ID, Flags, Event Type, Timestamp, and Payload Length.
3. Validate the Version field to ensure compatibility.
4. Read the payload based on the Payload Length.
5. Verify that parsing the TLVs consumes exactly Payload Length bytes (no leftover or short read).
6. Parse the payload by reading each TLV field and processing known field types.
7. Build Canonical Payload Bytes (see Canonicalization) and, if present, verify HMAC (0x10) first using constant-time comparison. Drop the packet if verification fails.
8. If Signature (0x12) is present, verify the signature using Public Key (0x13) or by resolving Auth Key ID (0x18) in a local trust store keyed by (NODE ID, Auth Key ID). Drop the packet if verification fails.
9. Apply replay protection: reject if Message ID has been seen before from the same NODE ID within REPLAY_WINDOW (see Replay Cache).
10. Process the event.

### 6.3 Parsing and validation rules
- Reject packet if:
  - Payload Length exceeds [MAX_PAYLOAD_SIZE](#max_payload_size) or actual payload bytes do not match length.
  - Any TLV overruns the payload buffer (Length too large) or nested sub-TLVs overrun their parent Length.
  - Duplicate singleton TLVs occur: 0x10, 0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18 (unless explicitly allowed by future versions). Data TLVs (0x01–0x05) MAY repeat.
  - HMAC (0x10) appears after Signature (0x12) or Signature is not the last field.
  - Flags contain non-zero reserved bits for version 1.
  - Total TLV count (including sub-TLVs) exceeds [MAX_TLV_COUNT](#max_tlv_count).
- Unknown TLV Types MUST be ignored (skipped) but still contribute to canonicalization ordering if present.
- HMAC verification MUST use constant-time comparison.

# 7. Replay cache (operational guidance)
- A receiver MUST maintain a replay cache keyed by (NODE ID 0x14, Message ID header 4 bytes).
- Entries SHOULD expire after [REPLAY_WINDOW](#replay_window). Implementations MAY cap cache size and evict oldest entries when limits are reached.
- Message ID MUST be cryptographically random. Reuse within REPLAY_WINDOW MUST be treated as replay.

# 8. Minimum Interop Profile v1
A smallest common set for verifiable events on constrained networks:
- Required TLVs:
  - Event Name (0x01)
  - NODE ID (0x14)
  - Auth Key ID (0x18) — exactly 4 bytes
  - One of: HMAC (0x10) or Signature (0x12)
- Ordering:
  - Data TLVs sorted by Type ascending, then security TLVs: [Public Key (0x13) if needed] -> Auth Key ID (0x18) -> HMAC (0x10) -> Signature (0x12, last)
- Key lookups: Receivers resolve keys from a local trust store keyed by (NODE ID, Auth Key ID). Public Key (0x13) is only needed if the receiver lacks a key for this pair.

# Variables
### MAX_PACKET_SIZE
* **Value**: 548 (RFC 791 Minimum MTU(576) - IP(20) and UDP(8) headers) bytes
* **Description**:
This size is chosen to ensure that packets can be transmitted over networks with a minimum MTU without fragmentation, which is important for maintaining low latency and high reliability in event delivery.

### MAX_PAYLOAD_SIZE
* **Value**: 531 (MAX_PACKET_SIZE(548) - headers(17)) bytes
* **Description**:
This allows for the fixed-size header while keeping the total packet size within the recommended limit of [MAX_PACKET_SIZE](#max_packet_size).

### REPLAY_WINDOW
* **Value**: 300 seconds (5 minutes)
* **Description**:
This window is suggested to balance between allowing for some network delay and clock skew, while still providing effective protection against replay attacks.

### CLOCK_SKEW_ALLOWANCE
* **Value**: 120 seconds (2 minutes)
* **Description**:
This allowance is recommended to account for differences in system clocks between NODEs, ensuring that legitimate messages are not rejected due to minor time discrepancies.

### MAX_TLV_COUNT
* **Value**: 64 (RECOMMENDED)
* **Description**:
Parsers MUST reject packets that contain more than this number of TLVs (including sub-TLVs) to bound worst-case parsing cost and mitigate abuse.
