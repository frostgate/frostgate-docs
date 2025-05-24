# FrostMessage: Canonical Cross-Chain Message Format

The `FrostMessage` struct is the **core data structure** for representing cross-chain messages in the Frostgate protocol. It encapsulates all information required for secure, verifiable, and interoperable message passing between heterogeneous blockchains.

---

## **Why FrostMessage?**

- **Standardization:** Provides a unified format for all cross-chain messages, regardless of source or destination chain.
- **Security:** Includes cryptographic proofs and metadata for robust verification.
- **Extensibility:** Designed to support new fields and ZK proof systems as the protocol evolves.
- **Interoperability:** Enables seamless relaying, replay, and auditing across EVM, Substrate, Solana, and more.

---

## **Structure**

```rust
pub struct FrostMessage {
    pub id: Uuid,
    pub source_chain: ChainId,
    pub destination_chain: ChainId,
    pub payload: Vec<u8>,
    pub proof: Option<ProofData>,
    pub timestamp: u64,
    pub nonce: u64,
    pub sender: Option<Vec<u8>>,
    pub recipient: Option<Vec<u8>>,
    pub metadata: Option<MessageMetadata>,
}
```

---

## **Field Descriptions**

- **`id: Uuid`**  
  Globally unique identifier for the message. Used for deduplication, tracking, and replay protection.

- **`source_chain: ChainId`**  
  Identifier for the originating blockchain.

- **`destination_chain: ChainId`**  
  Identifier for the target blockchain.

- **`payload: Vec<u8>`**  
  Opaque application data to be delivered cross-chain. Can represent contract calls, asset transfers, governance actions, etc.

- **`proof: Option<ProofData>`**  
  Optional cryptographic proof (e.g., ZK proof, signature, Merkle proof) attesting to the message’s validity or inclusion.

- **`timestamp: u64`**  
  Unix timestamp (seconds) when the message was created or submitted.

- **`nonce: u64`**  
  Monotonically increasing value for replay protection and ordering.

- **`sender: Option<Vec<u8>>`**  
  Optional sender address or public key (chain-specific encoding).

- **`recipient: Option<Vec<u8>>`**  
  Optional recipient address or public key (chain-specific encoding).

- **`metadata: Option<MessageMetadata>`**  
  Optional extensible metadata (e.g., fee info, routing hints, custom fields).

---

## **Usage Patterns**

- **Message Submission:**  
  Construct a `FrostMessage` with all required fields and submit via a `ChainAdapter`.

- **Proof Attachment:**  
  Attach a ZK proof or signature in the `proof` field for verifiable relaying.

- **Auditing & Replay:**  
  Use `id`, `timestamp`, and `nonce` for tracking, replay protection, and audit trails.

- **Custom Metadata:**  
  Extend `metadata` for application-specific needs (e.g., fee markets, priorities).

---

## **Example**

```rust
let msg = FrostMessage {
    id: Uuid::new_v4(),
    source_chain: ChainId::Ethereum,
    destination_chain: ChainId::Polkadot,
    payload: b"transfer:0x1234...".to_vec(),
    proof: Some(ProofData::Zk(ZkProof { /* ... */ })),
    timestamp: 1_717_000_000,
    nonce: 42,
    sender: Some(sender_address_bytes),
    recipient: Some(recipient_address_bytes),
    metadata: Some(MessageMetadata::default()),
};
```

---

## **Best Practices**

- **Always set a unique `id`** for every message.
- **Include a `proof`** whenever possible for trust-minimized relaying.
- **Use `nonce` and `timestamp`** for ordering and replay protection.
- **Leverage `metadata`** for extensibility and future-proofing.

---

## **Related Types**

- [`ChainId`](./chainid.md): Enum of supported blockchains.
- [`ProofData`](./proofdata.md): Enum for ZK, signature, or Merkle proofs.
- [`MessageMetadata`](./messagemetadata.md): Extensible metadata for advanced use cases.

---

## **FAQ**

**Q: Can I use custom payload formats?**  
A: Yes! The `payload` is opaque; encode your application data as needed.

**Q: Is the proof required?**  
A: No, but including a proof is strongly recommended for secure, trust-minimized relaying.

**Q: How do I prevent replay attacks?**  
A: Use unique `id` and incrementing `nonce` per sender.

---

## **See Also**

- [`ChainAdapter`](./chainadapter.md)
- [`ProofData`](./proofdata.md)
- [`ZkPlug`](./zkplug.md)
- [Frostgate SDK API Docs](../src/lib.rs)

---

**FrostMessage: The universal, verifiable message format for modular, cross-chain communication.**