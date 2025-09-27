# /// ***Message frame protocol***

---

# 1. Introduction

## 1.1 Purpose and Scope
This specification defines a transport-agnostic message framing protocol designed for secure, extensible, and interoperable communication across distributed systems. It provides a canonical structure for encoding messages, enforcing integrity, and supporting future evolution through governed extensions.

## 1.2 Design Goals
The protocol is built around the following principles:
- **Zero Trust Posture:** All inputs are treated as untrusted. Parsers must validate every field strictly and reject malformed or ambiguous frames.
- **Extensibility:** The frame format supports optional TLV-based extensions with a formal registry and governance model.
- **Transport Independence:** Frames are designed to be embedded in any byte stream, with no assumptions about underlying transport semantics.
- **Cryptographic Integrity:** Support for Ed25519 signatures and AEAD encryption ensures authenticity and confidentiality.
- **Compression-Aware:** Optional zstd compression is supported for payload efficiency.
- **Versioned Evolution:** The protocol includes mechanisms for version negotiation and extension lifecycle management.

## 1.3 Architectural Overview
At its core, the protocol defines a fixed-length header followed by a variable-length body. The header includes metadata for routing, validation, and interpretation. The body may contain plaintext, encrypted payloads, or structured extensions.

```text
+----------------+------------+----------------+----------------+
|     Header     | Extensions |      Body      |   Signature    |
+----------------+------------+----------------+----------------+
```

Frames are self-contained and designed for forward compatibility. Parsers must ignore unknown extensions and enforce strict validation on known fields.

# 2. Frame Anatomy

## 2.1 Frame Layout
The header is a fixed-size structure with the following fields:

| Field             | Size | Description                                                   | 
|-------------------|------|---------------------------------------------------------------| 
| Magic             | 6    | High-entropy sync marker, implementation-defined constant     | 
| Version           | 1    | Protocol version (v1 = 0x10)                                  | 
| Message ID        | 16   | Unique identifier (producer-scoped)                           | 
| Header Len        | 2    | Total header length (bytes), excluding extensions and payload | 
| Header Version    | 1    | Header layout version (v1 = 0x01)                             | 
| Frame Type        | 1    | Semantic type: 0x01=data, 0x02=ack, 0x03=error, 0x04=control  | 
| Flags             | 1    | Bitfield for encryption/compression/extension containers      | 
| Payload Type      | 1    | 0x01=UTF-8, 0x02=CBOR, 0x03=Opaque (encrypted), 0x04=Binary   | 
| Payload Len       | 4    | Length of payload (bytes)                                     | 
| Timestamp         | 8    | Unix time in milliseconds                                     | 
| Header CRC.       | 4.   | CRC-32 (IEEE 802.3) over Magic through Timestamp                       | 
| Extension Flags   | 1    | Per-extension policy flags (see below)                        | 
| Extension Count   | 1    | Number of TLV-style extensions                                | 
| Extensions        | var  | Ordered TLVs (see registry)                                   |
| Extension CRC     | 4    | CRC-32 (IEEE 802.3) over Extension region                        | 
| Payload           | var  | Message content (plaintext or ciphertext)                     |
| Payload CRC       | 4    | CRC-32 (IEEE 802.3) over payload bytes                        |
| Message Signature | 64   | Ed25519 signature over signed scope                           |
| Padding           | var  | Optional to align to 64B boundary                             |

<details>
<summary>Extended descriptions:</summary>

- **Magic:** A fixed 6-byte sequence. Suggested **0x3A7F21C9D4B8** to allow resynchronization in streams. Must be unique and high-entropy.
  - Rationale: Enables mid-stream recovery and detection of desynchronization.
  - Servers MAY choose a different value but MUST document it.
  - Parsers MUST scan for this value to align on frame boundaries.
- **Version:** Protocol version. Current is 0x10 (v1). Future versions will increment this byte.
  - Rationale: Allows for backward-compatible evolution.
  - Parsers MUST reject unsupported versions.
  - First 4 bits = major, last 4 bits = minor. (e.g., 0x11 = v1.1)
- **Message ID:** 16-byte unique identifier for the message, scoped to the producer. Recommended to use UUIDv4 or similar.
  - Rationale: Enables deduplication, tracing, and correlation.
  - MUST be unique per producer; collisions may lead to undefined behavior.
- **Header Len:** Total length of the header in bytes, excluding extensions and payload. Minimum is 40 bytes (fixed fields).
  - Rationale: Allows parsers to skip unknown extensions
  - MUST be at least 40.
- **Header Version:** Layout version of the header. Current is 0x01.
  - Rationale: Allows for header evolution independent of protocol version.
  - Parsers MUST reject unsupported header versions.
- **Frame Type:** Semantic type of the frame. See table for defined values.
  - Rationale: Enables parsers to interpret the payload correctly.
  - MUST be one of the defined types; unknown types cause rejection.
- **Flags:** Bitfield indicating encryption, compression, and extension policies. See below for bit definitions.
  - Rationale: Provides flexibility in framing options.
  - MUST be interpreted as a bitfield; reserved bits MUST be zero.
- **Payload Type:** Indicates the encoding of the payload. See table for defined values.
  - Rationale: Enables parsers to decode the payload correctly.
  - MUST be one of the defined types; unknown types cause rejection.
- **Payload Len:** Length of the payload in bytes. Zero if no payload.
  - Rationale: Allows parsers to read the correct amount of data.
  - MUST be accurate; parsers MUST reject frames where the actual payload length does not match this field.
- **Timestamp:** Unix time in milliseconds since epoch (1970-01-01T00:00:00Z).
  - Rationale: Enables time-based processing and replay detection.
  - MUST be in UTC; parsers MUST reject timestamps too far in the future. Configurable skew (e.g., ±5 minutes) is RECOMMENDED.
  - Receivers MAY use this for logging, ordering, or TTL enforcement.
- **Header CRC**
- **Extension Flags:** Bitfield indicating per-extension policies (e.g., criticality, encryption, compression). See below for bit definitions.
  - Rationale: Provides granular control over extension handling.
  - MUST be interpreted as a bitfield; reserved bits MUST be zero.
  - Parsers MUST enforce criticality and encryption flags as specified.
  - Unknown non-critical extensions MUST be ignored; unknown critical extensions MUST cause rejection.
- **Extension Count:** Number of TLV-style extensions present. Zero if no extensions.
  - Rationale: Allows parsers to know how many extensions to expect.
  - MUST be accurate; parsers MUST reject frames where the actual number of extensions does not match this field.
- **Extensions:** Ordered list of TLV-style extensions. See registry for defined types.
  - Rationale: Enables extensibility and future-proofing.
  - MUST be ordered by ascending Type; parsers MUST reject frames with out-of-order extensions.
  - Unknown non-critical extensions MUST be ignored; unknown critical extensions MUST cause rejection.
- **Extension CRC:** CRC-32 (IEEE 802.3) checksum over the extension region (from Extension flags through Extensions).
  - Rationale: Detects corruption in the header.
  - MUST be validated before processing the frame further.
  - Parsers MUST reject frames with invalid CRC.
  - Polynomial 0x04C11DB7, init 0xFFFFFFFF, reflected input/output, xorout 0xFFFFFFFF.
- **Payload:** The message content, either plaintext or ciphertext depending on Flags.
  - Rationale: Carries the actual data being transmitted.
  - MUST be interpreted according to Payload Type and Flags.
- **Payload CRC:** CRC-32 (IEEE 802.3) checksum over the payload bytes.
  - Rationale: Detects corruption in the payload.
  - MUST be validated before processing the payload further.
  - Parsers MUST reject frames with invalid CRC.
  - Polynomial 0x04C11DB7, init 0xFFFFFFFF, reflected input/output, xorout 0xFFFFFFFF.
- **Message Signature:** Ed25519 signature over the signed scope (from Magic through Payload CRC).
  - Rationale: Ensures authenticity and integrity of the frame.
  - All frames MUST be signed.
  - MUST be verified using the public key corresponding to the Identity TLV (0x12) if present.
  - Parsers MUST reject frames with invalid signatures.
  - MUST be exactly 64 bytes.
- **Padding:** Optional bytes to align the frame to a 64-byte boundary. Value is undefined and MUST be ignored.
  - Rationale: Improves performance on some transports and storage systems.
  - MUST be zero or more bytes; parsers MUST skip these bytes after processing the signature.
  - Alignment is RECOMMENDED but not required.
  - Padding MUST only contain 0x00 bytes.

</details>`

### Frame type

* **0x01 — Data:** Carries application data in the payload.
* **0x02 — Acknowledgment:** Acknowledges receipt of a data frame;
* **0x03 — Error:** Indicates an error condition; payload may contain error details.
* **0x04 — Control:** Used for control messages (e.g., heartbeats, session management).
* **0x05–0xFF:** Reserved for future use; parsers MUST reject unknown types.

### Header flags

* **Bit 0 (0x01):** Payload encrypted (AEAD).
* **Bit 1 (0x02):** Full-frame encrypted (AEAD) excluding always-clear fields.
* **Bit 2 (0x04):** Extensions may contain encrypted TLV containers.
* **Bit 3 (0x08):** Payload compressed (zstd) prior to encryption.
* **Bits 4–7:** Reserved, MUST be zero in v1.

### Payload types
* **0x01 — UTF-8:** Payload is a UTF-8 encoded string.
* **0x02 — CBOR:** Payload is encoded in CBOR format.
* **0x03 — Opaque:** Payload is opaque binary data (e.g., encrypted blob).
* **0x04 — Binary:** Payload is raw binary data.
* **0x05–0xFF:** Reserved for future use; parsers MUST reject unknown types.

### Extension flags (per TLV block)

* **Bit 0 (0x01):** Critical extension; unknown types MUST cause rejection.
* **Bit 1 (0x02):** Extension value is encrypted (AEAD over TLV value).
* **Bit 2 (0x04):** Extension value compressed (zstd).
* **Bits 3–7:** Reserved, MUST be zero in v1.

## 2.2 Encoding rules (endianness, alignment, padding)
* **Endianness:** All multi-byte integers are encoded in big-endian (network byte order).
* **Alignment:** The header is aligned to 4-byte boundaries; padding is added at the end of the frame to align to 64-byte boundaries if necessary.
* **Padding:** Padding bytes MUST be zero (0x00) and are not included in CRC or signature calculations.
* **Length fields:** Length fields (Header Len, Payload Len) MUST accurately reflect the actual byte counts; parsers MUST reject frames where lengths do not match actual data.

## 2.3 Reserved bits and future-proofing
* **Reserved fields:** Reserved bits in Flags and Extension Flags MUST be zero in v1 and ignored by parsers. Future versions may define new semantics. Parsers Must reject frames with non-zero reserved bits.
* **Versioning:** Major version increments (e.g., 0x10 to 0x20) indicate breaking changes; minor increments (e.g., 0x10 to 0x11) indicate backward-compatible additions.
* **Extension handling:** Parsers MUST ignore unknown non-critical extensions and MUST reject frames with unknown critical extensions.

# 3. Core Semantics
## 3.1 Frame types and their semantics

Regardless of frame type, the following general rules apply:
- **Integrity checks:** All frames MUST pass header and payload CRC checks before further processing.

### 3.1.1 Data frames
Data frames are the main carrier of application payloads. This protocol defers semantic interpretation of payloads to the application layer. The following guidelines apply:
- **Payload encoding:** The Payload Type field indicates how to interpret the payload (e.g., UTF-8, CBOR). Applications MUST agree on the expected encoding.
- **Size limits:** While the protocol supports large payloads (up to 4 GiB), applications MAY impose lower limits based on use case and transport capabilities.
- **Fragmentation:** The protocol does not define fragmentation or reassembly. Applications MUST handle this at a higher layer if needed.
- **Acknowledgments:** Data frames MAY be acknowledged using acknowledgment frames (Type 0x02). The application layer defines acknowledgment semantics (e.g., per-message, cumulative).
- **Error handling:** Applications MUST define how to handle errors indicated by error frames (Type 0x03). The payload of error frames MAY contain diagnostic information.
- **Compression and encryption:** Data frames MAY be compressed and/or encrypted based on the Flags field. Applications MUST ensure that both sender and receiver agree on these settings.
- **Ordering:** Data frames MAY be sent out of order. Applications MUST handle ordering if required by the use case.

### 3.1.2 Acknowledgment frames
Acknowledgment frames are used to confirm receipt of other frame types. The following guidelines apply:
- **MessageID payload:** Acknowledgment frames carry only the Message ID of the acknowledged frame in the payload. The Payload Type MUST be set to Binary (0x04).
- **No payload:** Acknowledgment frames MUST NOT contain any additional payload beyond the Message ID.
- **Idempotency:** Acknowledgment frames are idempotent; receiving duplicate acknowledgments for the same Message ID MUST have no additional effect.
- **Payload length:** The Payload Len MUST be exactly 16 bytes (size of Message ID).
- **Error handling:** If an acknowledgment frame is received for a Message ID that was not sent or has already been acknowledged, it MAY be ignored or logged as a warning.
- **Compression and encryption:** Acknowledgment frames MAY be compressed and/or encrypted based on the Flags field. Both sender and receiver MUST agree on these settings.
- **Timing:** Acknowledgment frames SHOULD be sent promptly after receiving the corresponding data frame to minimize latency. RECOMMENDED within 100ms of receipt.
- **Ordering:** Acknowledgment frames MAY be sent out of order relative to the data frames they acknowledge. Applications MUST handle this appropriately.

### 3.1.3 Error frames
Error frames indicate error conditions encountered during processing. All error frames have the payload type 0x01. Error frames have the following Extensions defined:

- **Core 0x11 (18) - Error**
  - **Code (1B):** Numeric error code (e.g., 0x0001=Bad signature, 0x0002=Invalid CRC, 0x0003=Unknown critical extension).
  - **Message (var):** Optional UTF-8 string with human-readable error description.
  - **Rule:** MUST be present in error frames; parsers MUST log or surface this information for diagnostics.

<details>

<summary>Defined error codes:</summary>

| Error Code | Error text              | Description                                                 |
|------------|-------------------------|-------------------------------------------------------------|
| 0x01       | BAD_SIGNATURE           | Signature verification failed.                              |
| 0x02       | INVALID_PAYLOAD_CRC     | Header or payload CRC check failed.                         |
| 0x03       | UNKNOWN_EXTENSION       | Unknown critical extension encountered.                     |
| 0x04       | MALFORMED               | Frame structure is malformed.                               |
| 0x05       | UNSUPPORTED             | Unsupported version or type.                                |
| 0x06       | REPLAY                  | Frame rejected due to replay detection.                     |
| 0x07       | DECRYPT_FAIL            | AEAD decryption failed.                                     |
| 0x08       | TIMEOUT                 | Frame processing timed out.                                 |
| 0x09       | POLICY_VIOL             | Frame violates local policy.                                |
| 0x0A       | INTERNAL_ERR            | Internal processing error.                                  |
| 0x0B       | NOT_AUTHED              | Sender not authenticated.                                   |
| 0x0C       | NO_IDENTITY             | Missing required identity extension.                        |
| 0x0D       | KEY_EXPIRED             | Key epoch is expired.                                       |
| 0x0E       | PAYLOAD_TOO_LARGE       | Payload exceeds allowed size.                               |
| 0x0F       | INVALID_TIMESTAMP       | Timestamp is out of acceptable range.                       |
| 0x10       | UNKNOWN_TYPE            | Unknown frame type.                                         |
| 0x11       | INVALID_PAYLOAD         | Payload does not conform to expected format.                |
| 0x12       | COMPRESSION_ERR         | Compression or decompression failed.                        |
| 0x13       | EXTENSION_ERR           | Error processing extensions.                                |
| 0x14       | SESSION_ERR             | Session management error.                                   |
| 0x15       | RATE_LIMITED            | Sender is rate-limited.                                     |
| 0x16       | RESOURCE_EXHAUSTED      | Insufficient resources to process frame.                    |
| 0x17       | NOT_IMPLEMENTED         | Feature not implemented.                                    |
| 0x18       | UNAUTHORIZED            | Unauthorized access attempt.                                |
| 0x19       | INVALID_HEADER_CRC      | Header CRC is invalid.                                      |
| 0x1A       | INVALID_FLAGS           | Flags field contains invalid values.                        |
| 0x1B       | INVALID_EXT_COUNT       | Extension count does not match actual number of extensions. |
| 0x1C       | INVALID_HEADER_LEN      | Header length field is incorrect.                           |
| 0x1D       | INVALID_PAYLOAD_LEN     | Payload length field is incorrect.                          |
| 0x1E       | INVALID_MAGIC           | Magic value does not match expected.                        |
| 0x1F       | UNKNOWN_ERROR           | An unknown error occurred.                                  |
| 0x20       | TIME_SYNC_ERR           | Time synchronization error.                                 |
| 0x21       | BAD_IDENTITY            | Identity extension is invalid or does not match signature.  |
| 0x22       | KEY_MISMATCH            | Key epoch does not match expected value.                    |
| 0x23       | REPLAY_STORE_FULL       | Replay detection store is full.                             |
| 0x24       | INVALID_REPLAY          | Replay detection parameters are invalid.                    |
| 0x25       | COMPRESSION_UNSUPPORTED | Compression algorithm is unsupported.                       |
| 0x26       | ENCRYPTION_UNSUPPORTED  | Encryption algorithm is unsupported.                        |
| 0x27       | SIGNATURE_UNSUPPORTED   | Signature algorithm is unsupported.                         |
| 0x28       | INVALID_MESSAGE_ID      | Message ID is malformed.                                    |
| 0x29       | PAYLOAD_MISMATCH        | Payload does not match expected content.                    |
| 0x2A       | EXTENSION_MISMATCH      | Extension content does not match expected format.           |
| 0x2B       | INVALID_TIMESTAMP_FMT   | Timestamp format is invalid.                                |

> 0x2C–0x9F are reserved for future use.
> 
> 0xA0–0xFF are application-defined error codes.

</details>

## 3.2 Control flow (e.g. handshake, session, heartbeat)
Control flow is deferred to the application layer. However, the following guidelines apply:
- **Session management:** Control frames (Type 0x04) MAY be used for session initiation, termination, and heartbeats. The application layer MUST define the semantics.
- **Heartbeats:** If used, heartbeat frames SHOULD be sent at regular intervals (e.g., every 30 seconds) to maintain session liveness. The application layer MUST define the expected response behavior.
- **Session state:** The protocol does not define session state management. Applications MUST handle session state, including key management and replay detection, at a higher layer.

## 3.3 Error signaling and recovery
Error frames (Type 0x03) are used to signal error conditions. The following guidelines apply:
- **Error reporting:** When a parser encounters an error (e.g., invalid CRC, bad signature), it MUST generate an error frame with the appropriate error code and optional message.
- **Recovery:** Upon receiving an error frame, the application layer MUST determine the appropriate recovery action (e.g., retransmission, session termination).
- **Logging:** Parsers SHOULD log error conditions for diagnostics and auditing purposes.
- **Rate limiting:** To prevent denial-of-service attacks, parsers MAY implement rate limiting on error frame generation.
- **Unknown errors:** If an unknown error condition is encountered, the parser MUST use the UNKNOWN_ERROR code (0x001F) and provide as much context as possible in the message field.

# 4. Validation and Parsing
## 4.1 Parser behavior and strict validation rules

* **Strict validation:** Parsers MUST reject ambiguous or malformed frames before signature verification.
* **Canonical ordering:** Extensions MUST be ordered by ascending Type; deviations cause rejection.
* **Unknown non-critical extensions:** Parsers MUST ignore and preserve order; bytes remain within the signature scope.
* **Unknown critical extensions:** Parsers MUST reject the frame; responders MAY send an Error codes frame indicating UnknownExtension.
* **Error reporting:** When feasible, parsers SHOULD emit Error codes TLVs on rejection paths, including Frame ref if available.
* **Resynchronization:** On error (CRC failure, malformed structure), parsers MUST discard bytes until the next valid Magic is found.
* **Strict parsing:** Reject frames early on CRC or structural errors; do not attempt heuristic repairs beyond Magic resynchronization.
* **Deterministic ordering:** Produce canonical TLV order; receivers SHOULD warn on deviations.
* **Interoperability suite:** Maintain fixtures with hex frames, computed CRC-32 values, AEAD nonces/tags, and RFC 8032 signatures across multiple languages.
* **Observability:** Log Error codes TLVs on rejection paths; include Frame ref when available for correlation.

## 4.2 Replay detection and integrity checks

* **Replay bounds:** After successful AEAD and signature verification, parsers MUST check the replay store under the configured window. If the semantic digest is present and falls within the window, reject on duplicate. Insert digest upon acceptance with appropriate TTL or eviction metadata.
* **Eviction timing:** Eviction MAY be time-driven (wheel/buckets) or occurrence-driven (LRU per identity). Implementations MUST avoid unbounded growth.
* **Replay store design:** Prefer sharded probabilistic sets per sender fingerprint with a small time-bucket wheel (e.g., 60 buckets × 1,000 ms). On bucket expiry, drop the entire bucket to amortize deletion cost.
* **False positive management:** Choose FPR to balance memory and false rejects; for critical control frames, consider dual-check: probabilistic membership followed by a small LRU exact set for the last M digests (e.g., M=1024).
* **Memory budgeting:** Implementations SHOULD cap total memory for replay detection and degrade gracefully (increase FPR or shrink TTL) under pressure.
* **Defaults:** TTL_ms=900,000; FPR=1e-4; Max entries per identity=1,000,000; Shard granularity=1,000 ms; Filter type=Cuckoo (software), Xor (memory-efficient).
* **Semantic hash:** Use SHA-256 over the payload and all critical extensions (excluding non-critical unknowns). The hash is included as a TLV (0x1B) in the extensions. Parsers MUST compute and verify this hash before replay checks.
* **Metrics:** Maintain counters for accepted frames, rejected frames (by error code), replay rejections, and memory usage for replay detection.

## 4.3 Canonical parsing steps and edge cases

1. **Sync on Magic:** Scan stream to align on the 6-byte Magic.
2. **Read envelope:** Read 4-byte length prefix and then the indicated frame bytes.
3. **Read fixed header:** Parse fields through Header Len and Payload Len.
4. **Validate Header CRC-32:** Compute and compare.
5. **Parse extension control:** Read Payload Type, Extension Flags, Extension Count.
6. **Parse TLVs:** Enforce ordering, apply per-TLV encryption/compression as indicated.
7. **Read payload:** Respect Payload Len.
8. **Validate Payload CRC-32:** Compute and compare.
9. **Verify signature:** Ed25519 over the signed scope.
10. **Verify AEAD:** If encryption is present, validate AEAD tag with AAD.
11. **Replay checks:** Apply semantic hash and replay window policy.
12. **Skip padding:** Align to 64B boundary if present.
13. **Process frame:** Deliver to application layer based on Frame Type.

# 5. Cryptographic Integrity
## 5.1 Signature and verification rules (Ed25519)

* **Signature scope:** Ed25519 signature covers from Magic through Payload CRC, including all extensions.
* **Key management:** Public keys are identified via the Identity TLV (0x12).
* **Signature verification:** Parsers MUST verify the signature using the public key corresponding to the Identity TLV. If verification fails, the frame MUST be rejected with an appropriate error code.
* **Key formats:** Public keys MUST be in raw 32-byte format as per RFC 8032. Private keys are out of scope for this specification.
* **Signature generation:** Senders MUST generate signatures using the private key corresponding to the Identity TLV. The signature MUST be exactly 64 bytes.
* **Algorithm:** The signature algorithm is Ed25519 as defined in RFC 8032. No other signature algorithms are supported in v1.
* **Key rotation:** Key epochs (0x16 TLV) MAY be used to indicate key rotation. Parsers MUST reject frames with expired key epochs based on local policy.
* **Error handling:** If signature verification fails, parsers MUST generate an error frame with the BAD_SIGNATURE code (0x0001).
* **Performance considerations:** Implementations SHOULD optimize signature verification for high-throughput scenarios, including batch verification if supported by the cryptographic library.
* **Security considerations:** Private keys MUST be stored securely and never transmitted. Public keys SHOULD be distributed through secure channels or trusted registries.

## 5.2 AEAD encryption framing

* **Encryption scope:** AEAD encryption covers the payload and any TLV values marked as encrypted in the Extension Flags. The AAD includes all always-clear fields (Magic through Payload CRC).
* **Algorithms:** Chacha20-Poly1305 and AES-256-GCM are supported. The algorithm is indicated in the Identity TLV (0x12) or a dedicated TLV if needed.
* **Key management:** Encryption keys are out of scope for this specification. Implementations MUST ensure secure key distribution and storage.
* **Nonce management:** Nonces MUST be unique per key and session. The Encrypted nonce TLV (0x24) MUST be used to convey the AEAD nonce.
* **Tag length:** The AEAD tag MUST be 16 bytes as per AES-GCM specification.
* **Encryption process:** Senders MUST encrypt the payload and any encrypted TLV values using the AEAD algorithm with the specified key and nonce. The encrypted data replaces the plaintext in the frame.
* **Decryption process:** Parsers MUST decrypt the payload and any encrypted TLV values using the AEAD algorithm with the specified key and nonce. If decryption fails, the frame MUST be rejected with an appropriate error code.
* **Error handling:** If decryption fails, parsers MUST generate an error frame with the DECRYPT_FAIL code (0x0007).
* **Performance considerations:** Implementations SHOULD optimize encryption and decryption for high-throughput scenarios,

## 5.3 Key epochs and rollover

* **Epoch definition:** Key epochs are defined using the Key epoch TLV (0x16). The epoch is a 4-byte unsigned integer that increments with each key rotation.
* **Rollover policy:** Implementations MUST define a key rollover policy, including cadence and grace periods. Parsers MUST reject frames with expired key epochs based on local policy.
* **Epoch validation:** When the Key epoch TLV is present, parsers MUST validate that the epoch is current. If the epoch is expired, the frame MUST be rejected with the KEY_EXPIRED code (0x000D).
* **Grace periods:** Implementations MAY define grace periods during which frames with the previous epoch are still accepted. The specifics of the grace period are deployment-specific and out of scope for this specification.
* **Key distribution:** Key distribution and management are out of scope for this specification. Implementations MUST ensure secure key distribution and storage.
* **Error handling:** If a frame is rejected due to an expired key epoch, parsers MUST generate an error frame with the KEY_EXPIRED code (0x000D).
* **Security considerations:** Key epochs help mitigate the risk of key compromise by limiting the window of exposure. Implementations MUST ensure that key rotation is performed securely and that old keys are retired appropriately.
* **Interoperability:** All implementations MUST support key epochs to ensure interoperability in environments where key rotation is required. including batch processing if supported by the cryptographic library.
* **Security considerations:** Nonces MUST be generated securely to prevent nonce reuse, which can compromise the security of the AEAD scheme.
* **Interoperability:** All implementations MUST support AEAD encryption to ensure interoperability in environments where confidentiality is required.

## 5.4 Security considerations

* **Signature coverage:** Frames are signed end-to-end; full-frame encryption signs ciphertext to ensure robust tamper resistance.
* **Compression before encryption:** Always compress plaintext payload before AEAD to avoid side-channel risks; do not compress identity or error TLVs.
* **Identity binding:** Identity TLV MUST match the key used for the signature; receivers MUST validate binding before trusting content.
* **Key rotation:** Use Key epoch TLV (0x16) to signal rotations with grace periods.
* **Replay policy:** Implement bloom filter or cache with parameters guided by Replay window TLV (0x23).
* **Error transparency:** Emit Error codes TLVs on rejection paths for diagnostics; include Frame ref when available.
* **Denial-of-service mitigation:** Rate-limit error frame generation to prevent amplification attacks.
* **Memory management:** Cap memory usage for replay detection; degrade gracefully under pressure.
* **Logging and auditing:** Maintain logs of error conditions and rejected frames for auditing and forensic analysis.
* **Private key security:** Private keys MUST be stored securely and never transmitted. Public keys SHOULD be distributed through secure channels or trusted registries.
* **Algorithm agility:** While v1 specifies Ed25519 and AES-256-GCM, future versions MAY introduce additional algorithms. Implementations SHOULD be designed with algorithm agility in mind.
* **Implementation security:** Implementations MUST follow best practices for secure coding, including input validation, error handling, and memory management to prevent vulnerabilities.
* **Transport security:** While this protocol provides end-to-end integrity and confidentiality, it is RECOMMENDED to use secure transport layers (e.g., TLS) for additional protection against network-level attacks.
* **Regular updates:** Implementations SHOULD be regularly updated to address newly discovered vulnerabilities and to stay current with cryptographic best practices.
* **Compression side-channels:** Compressing data before encryption helps mitigate side-channel risks associated with compression algorithms. Implementations MUST ensure that sensitive data is not exposed through compression artifacts. Sensitive TLVs (e.g., Identity, Error codes) MUST NOT be compressed.

# 6. Compression and Encoding
## 6.1 zstd framing and negotiation

* **Compression algorithm:** The compression algorithm is zstd as defined in RFC 8478. No other compression algorithms are supported in v1.
* **Compression scope:** Compression applies to the payload and any TLV values marked as compressed in the Extension Flags. The uncompressed length MUST be indicated in the Compression metadata TLV (0x22).
* **Compression process:** Senders MUST compress the payload and any compressed TLV values using the zstd algorithm before encryption (if encryption is applied). The compressed data replaces the original in the frame.
* **Decompression process:** Parsers MUST decompress the payload and any compressed TLV values using the zstd algorithm after decryption (if encryption is applied). If decompression fails, the frame MUST be rejected with an appropriate error code.
* **Error handling:** If decompression fails, parsers MUST generate an error frame with the COMPRESSION_ERR code (0x0012).
* **Compression levels:** The compression level is indicated in the Compression metadata TLV (0x22). Implementations MAY choose the level based on performance and resource considerations.
* **Interoperability:** All implementations MUST support zstd compression to ensure interoperability in environments where bandwidth optimization is required.
* **Performance considerations:** Implementations SHOULD optimize compression and decompression for high-throughput scenarios, including batch processing if supported by the compression library.
* **Security considerations:** Compressing data before encryption helps mitigate side-channel risks associated with compression algorithms. Implementations MUST ensure that sensitive data is not exposed through compression artifacts.
* **Negotiation:** Compression is negotiated implicitly through the presence of the Compression metadata TLV (0x22). Both sender and receiver MUST support zstd compression for frames that include this TLV.
* **Defaults:** Compression is optional; if the Compression metadata TLV is absent, the payload and TLV values are treated as uncompressed.

## 6.2 Optional compression strategies

* **Selective compression:** Implementations MAY choose to compress only certain types of payloads or TLV values based on content type, size, or other heuristics. For example, text-based payloads may benefit more from compression than already compressed binary data.
* **Compression thresholds:** Implementations MAY define size thresholds below which compression is not applied, as the overhead may outweigh the benefits. For example, payloads smaller than 128 bytes MAY be sent uncompressed.
* **Adaptive compression:** Implementations MAY adapt compression strategies based on network conditions, resource availability, or historical performance metrics. For example, if network latency is high, more aggressive compression levels MAY be used to reduce payload size.
* **Compression levels:** Implementations MAY allow configuration of compression levels based on performance and resource considerations. Higher compression levels provide better size reduction at the cost of increased CPU usage.
* **Content-aware compression:** Implementations MAY analyze payload content to determine the most effective compression strategy. For example, certain data formats (e.g., JSON, XML) may benefit from specific compression techniques.
* **Error handling:** If an implementation chooses to apply selective or adaptive compression, it MUST still adhere to the error handling guidelines outlined in section 6.1. If decompression fails, the frame MUST be rejected with the COMPRESSION_ERR code (0x0012).
* **Interoperability:** While optional compression strategies are implementation-specific, all implementations MUST support the baseline zstd compression as defined in section 6.1 to ensure interoperability.
* **Documentation:** Implementations that employ optional compression strategies SHOULD document their approach and any configuration options available to users.
* **Testing:** Implementations that use optional compression strategies SHOULD include test cases to validate the correctness of compression and decompression processes, especially when using adaptive or content-aware techniques.
* **Performance monitoring:** Implementations MAY include performance monitoring to assess the effectiveness of compression strategies and make adjustments as needed.
* **Security considerations:** Implementations MUST ensure that optional compression strategies do not introduce vulnerabilities or compromise the integrity and confidentiality of the data being transmitted.
* **Defaults:** If no optional compression strategies are employed, the implementation MUST fall back to the standard zstd compression as defined in section 6.1.

## 6.3 Interoperability notes

* **Baseline support:** All implementations MUST support the core features of the protocol, including frame structure, signature verification, AEAD encryption, and zstd compression as defined in sections 2 through 6.1.
* **Extension handling:** Implementations MUST correctly handle known extensions and MUST ignore unknown non-critical extensions while preserving their order. Unknown critical extensions MUST cause frame rejection.
* **Error reporting:** Implementations SHOULD emit Error codes TLVs on rejection paths for diagnostics, including Frame ref when available for correlation.

# 7. Extension System
## 7.1 TLV format and encoding rules

```text
+------------+------------+------------------+
| Type (1B)  | Length (3B)| Value (variable) |
+------------+------------+------------------+
```

* **Type (1B):** Identifies the extension type. High nibble indicates namespace (Core, Experimental, Vendor, Ephemeral), low nibble indicates specific type.
* **Length (3B):** Length of the Value field in bytes.
* **Value (variable):** Opaque byte sequence as defined by the specific extension type.

* **Ordering:** Canonical ascending by Type.
* **Coverage:** TLVs are included in the signature scope.
* **Namespaces:** High nibble is namespace, low nibble is type:
  * 0x1x Core (stable)
  * 0x2x Experimental (may change)
  * 0xAx Vendor (reserved ranges by vendor)
  * 0xEx Ephemeral (non-production)

> Vendor ranges (0xA0–0xBF) MUST register an owner and collision-avoidance rule (e.g., domain-hash prefix) in the registry. Governance is community-driven and documented elsewhere.
> Ephemeral types (0xE0–0xEF) are for local/testing only and MUST NOT be used in production.
> 
## 7.2 Extension registry and governance

| Extension Type | Namespace    | Description                 | Critical | Encrypted | Compressed |
|----------------|--------------|-----------------------------|----------|-----------|------------|
| 0x11           | Core         | Identity                    | Yes      | No        | No         |
| 0x12           | Core         | Device attestation          | No       | No        | No         |
| 0x13           | Core         | Signed scope digest         | No       | No        | No         |
| 0x14           | Core         | Key epoch                   | Yes      | No        | No         |
| 0x15           | Core         | Semantic hash               | No       | No        | No         |
| 0x16           | Core         | Compression metadata        | No       | No        | No         |
| 0x17           | Core         | Replay window               | No       | No        | No         |
| 0x18           | Core         | Encrypted nonce             | No       | No        | No         |
| 0x19           | Core         | Replay filter configuration | No       | No        | No         |
| 0x1A           | Core         | Padding                     | No       | No        | No         |
| 0x1B           | Core         | Error codes                 | Yes      | No        | No         |
| 0x1C           | Core         | AEAD Algorithm              | No       | No        | No         |
| 0x1D–0x1F      | Core         | Reserved                    | Varies   | Varies    | Varies     |            |
| 0x20–0x2F      | Experimental | Experimental extensions     | Varies   | Varies    | Varies     |
| 0xA0–0xBF      | Vendor       | Vendor-specific extensions  | Varies   | Varies    | Varies     |
| 0xE0–0xEF      | Ephemeral    | Local/testing extensions    | Varies   | Varies    | Varies     |

<details>
<summary>Extension registry</summary>

* **Identity (0x11):** Public key (32B) identifying the sender; critical for signature verification.
* **Device attestation (0x12):** Optional attestation data (var) for device authenticity.
* **Signed scope digest (0x13):** SHA-256 digest (32B) of the signed scope for quick verification.
* **Key epoch (0x14):** 4B unsigned integer indicating the key version.
* **Semantic hash (0x15):** SHA-256 hash (32B) over payload and critical extensions for replay detection.
* **Compression metadata (0x16):** 1B compression level + 4B uncompressed length.
* **Replay window (0x17):** 4B TTL in ms for replay detection.
* **Encrypted nonce (0x18):** 12B AEAD nonce for payload encryption.
* **Replay filter configuration (0x19):** 1B FPR exponent + 4B max entries + 4B shard granularity in ms.
* **Padding (0x1A):** Variable-length padding to align to 64B boundary.
* **Error codes (0x1B):** 2B error code + optional UTF-8 message for diagnostics.
* **AEAD Algorithm (0x1C):** 1B algorithm identifier (e.g., 0x01=Chacha20-Poly1305, 0x02=AES-256-GCM).
* **Reserved (0x1D–0x1F):** Reserved for future Core extensions.
* **Experimental (0x20–0x2F):** Community-defined extensions; may change.
* **Vendor (0xA0–0xBF):** Vendor-specific extensions; registration required.
* **Ephemeral (0xE0–0xEF):** Local/testing only; MUST NOT ship to production.

- Compression metadata
  - **Level (1B):** Compression level (0-22).
  - **Uncompressed length (4B):** Length of the uncompressed payload.
  - **Rule:** MUST be present if compression is applied; parsers MUST use this to allocate buffers.

</details>

## 7.3 Versioning and compatibility
### 7.3.1 Versioning

* **Namespace/type nibble:** High nibble = namespace, low nibble = type, e.g., 0x12 is Core:Identity.
* **Sub-versioning:** If a TLV requires evolution, include a version byte inside its Value; do not repurpose existing Type semantics.
* **Deprecation:** Mark deprecated types in the registry; parsers MUST still read them and SHOULD provide migration guidance.

### 7.3.2 Compatibility
* **Unknown non-critical extensions:** Parsers MUST ignore and preserve order; bytes remain within the signature scope.
* **Unknown critical extensions:** Parsers MUST reject the frame; responders MAY send an Error codes frame indicating UnknownExtension.
* **Extension evolution:** New types MUST NOT change the semantics of existing types. New functionality MUST be added via new types.
* **Interoperability:** Implementations MUST support all Core types and SHOULD support widely adopted Experimental types to ensure interoperability.
* **Testing:** New extension types MUST be accompanied by test vectors and interoperability tests to ensure correctness across implementations.
* **Documentation:** All extension types MUST be documented in the registry with clear semantics, usage guidelines, and examples.
* **Governance:** The extension registry is community-governed; proposals for new types or changes MUST follow the process outlined in section 7.4.2.

## 7.4 Reserved namespaces and lifecycle

### 7.4.1 Namespaces

* **Core (0x10–0x1F):** Stable, ratified after interop.
* **Experimental (0x20–0x2F):** Subject to change; not guaranteed backward compatibility.
* **Vendor (0xA0–0xBF):** Reserved ranges with owner registration and collision policy.
* **Ephemeral (0xE0–0xEF):** For local/testing only; MUST NOT ship to production.

### 7.4.2 Proposal process

1. **Draft:** Submit a spec PR with motivation, wire format, security analysis, and test vectors.
2. **Review:** Community and implementer review; at least two independent implementations.
3. **Ratify:** Assign a Core type and add to the living registry with conformance tests.
4. **Deprecation:** Mark deprecated types; parsers remain able to read them and SHOULD provide migration guidance.
5. **Removal:** After a deprecation period (e.g., 1 year), types MAY be removed from the registry; parsers MAY stop recognizing them.

# 8. Conformance and Interop
## 8.1 Canonical examples
- **Basic data frame:** Unencrypted, unsigned frame with a simple text payload.
- **Signed frame:** Frame with Identity TLV and valid Ed25519 signature.
- **Encrypted frame:** Frame with AEAD encryption applied to the payload.
- **Compressed frame:** Frame with zstd compression applied to the payload.
- **Full-featured frame:** Frame utilizing multiple extensions, including Identity, Key epoch, Semantic hash, and Replay window.
- **Error frame:** Frame demonstrating error signaling with an Error codes TLV.
- **Replay detection:** Example showing replay detection in action with duplicate frames.
- **Edge cases:** Frames with maximum payload size, minimum payload size, and various combinations of extensions.

## 8.2 Conformance test vectors
- **Test vector 1:** Basic data frame with expected byte sequence and CRC values.
- **Test vector 2:** Signed frame with expected signature and verification steps.
- **Test vector 3:** Encrypted frame with expected ciphertext and AEAD tag.
- **Test vector 4:** Compressed frame with expected compressed payload and decompression steps.
- **Test vector 5:** Full-featured frame with all extensions and expected behaviors.
- **Test vector 6:** Error frame with expected error code and message.
- **Test vector 7:** Replay detection scenario with expected acceptance and rejection of frames.
- **Test vector 8:** Edge case scenarios with expected handling of maximum and minimum payload sizes.

## 8.3 Implementation notes and edge cases
- **Memory management:** Implementations MUST manage memory efficiently, especially for replay detection stores. Use bounded data structures and avoid unbounded growth.
- **Performance optimization:** Implementations SHOULD optimize for high-throughput scenarios, including batch processing of frames and parallel cryptographic operations.
- **Error handling:** Implementations MUST handle errors gracefully, providing meaningful error codes and messages. Rate-limiting of error frame generation is RECOMMENDED to prevent denial-of-service attacks.
- **Logging and observability:** Implementations SHOULD provide logging of key events, including frame acceptance/rejection, error conditions, and replay detection events. Logs SHOULD include Frame ref when available for correlation.
- **Testing and validation:** Implementations MUST include comprehensive test suites covering all aspects of the protocol, including edge cases and error conditions. Interoperability testing with other implementations is RECOMMENDED.
- **Security best practices:** Implementations MUST follow security best practices, including secure key management, input validation, and protection against common vulnerabilities (e.g., buffer overflows, injection attacks).
- **Configuration flexibility:** Implementations SHOULD provide configuration options for key parameters, such as replay window size, compression levels, and cryptographic algorithms (where applicable).
- **Documentation:** Implementations MUST be well-documented, including configuration options, usage guidelines, and any deviations from the specification.

# 9. Appendices
## 9.1 Session and handshake notes
> Out of scope for this document. Implementations MAY use Noise NN/IK or other mutually authenticated key exchange protocols to establish shared secrets and session state (counters, epochs).

## 9.2 Reserved values and future extensions
- **Frame Types:** 0x05–0xFF reserved for future use.
- **Error Codes:** 0x002C–0x009F reserved for future use; 0x00A0–0x00FF application-defined.
- **Extension Types:** 0x1C–0x1F reserved; 0x20–0x2F Experimental; 0xA0–0xBF Vendor-specific; 0xE0–0xEF Ephemeral.
- **Cryptographic algorithms:** Future versions MAY introduce additional signature and encryption algorithms.
- **Compression algorithms:** Future versions MAY introduce additional compression algorithms.
- **Key management:** Future versions MAY define additional key management and rotation mechanisms.

## 9.3 Glossary of terms
- **AEAD:** Authenticated Encryption with Associated Data, a form of encryption that provides confidentiality, integrity, and authenticity.
- **AAD:** Associated Authenticated Data, data that is authenticated but not encrypted in AEAD schemes.
- **CRC-32:** A 32-bit cyclic redundancy check used for error detection in data transmission.
- **Ed25519:** A public-key signature system using the Edwards-curve Digital Signature Algorithm (EdDSA).
- **Frame:** A complete unit of data transmission in the protocol.
- **Magic:** A fixed byte sequence at the start of a frame used for synchronization.
- **Nonce:** A number used once, typically in cryptographic operations to ensure uniqueness.
- **Replay detection:** Mechanisms to prevent the acceptance of duplicate frames.
- **Semantic hash:** A hash computed over specific frame fields to uniquely identify the content for replay detection.
- **TLV:** Type-Length-Value, a common encoding scheme for optional data fields.
- **zstd:** Zstandard, a fast compression algorithm.
- **Key epoch:** A versioning mechanism for cryptographic keys to facilitate key rotation.
- **FPR:** False Positive Rate, the probability of incorrectly identifying a non-member as a member in probabilistic data structures.
- **Cuckoo filter:** A space-efficient probabilistic data structure for set membership testing.
- **Xor filter:** A space-efficient probabilistic data structure for set membership testing, optimized for memory usage.
- **LRU:** Least Recently Used, a cache eviction policy that removes the least recently accessed items first.
- **TTL:** Time To Live, a duration after which an entry is considered expired.
- **FIPS:** Federal Information Processing Standards, a set of standards for cryptographic modules.
- **RFC:** Request for Comments, a type of publication from the engineering and development community for the internet.
- **Interoperability:** The ability of different systems or implementations to work together seamlessly.
- **Namespace:** A categorization of TLV types to indicate stability and intended use.
