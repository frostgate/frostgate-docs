# Frostgate ChainAdapter: Unified Blockchain Integration Interface

The `ChainAdapter` trait is the core abstraction for integrating any blockchain into the Frostgate protocol. It provides a **unified, async, and extensible interface** for submitting, tracking, and verifying cross-chain messages, regardless of the underlying chain (EVM, Substrate, Solana, etc).

---

## **Why ChainAdapter?**

- **Abstraction:** Decouples Frostgate’s core logic from chain-specific details.
- **Extensibility:** Add new chains by implementing this trait—no changes to the protocol core.
- **Async & Production-Ready:** Designed for real-world, networked, and concurrent environments.
- **Testability:** Enables mocking and simulation for robust testing.

---

## **Trait Definition**

```rust
#[async_trait]
pub trait ChainAdapter: Send + Sync {
    type BlockId: Clone + std::fmt::Debug + Send + Sync + 'static;
    type TxId: Clone + std::fmt::Debug + Send + Sync + 'static;
    type Error: std::error::Error + Send + Sync + 'static;

    async fn latest_block(&self) -> Result<Self::BlockId, Self::Error>;
    async fn get_transaction(&self, tx: &Self::TxId) -> Result<Option<Vec<u8>>, Self::Error>;
    async fn wait_for_finality(&self, block: &Self::BlockId) -> Result<(), Self::Error>;
    async fn submit_message(&self, msg: &FrostMessage) -> Result<Self::TxId, Self::Error>;
    async fn listen_for_events(&self) -> Result<Vec<MessageEvent>, Self::Error>;
    async fn verify_on_chain(&self, msg: &FrostMessage) -> Result<(), Self::Error>;
    async fn estimate_fee(&self, msg: &FrostMessage) -> Result<u128, Self::Error>;
    async fn message_status(&self, id: &Uuid) -> Result<MessageStatus, Self::Error>;
    async fn health_check(&self) -> Result<(), Self::Error>;
}
```

---

## **Associated Types**

- **`BlockId`**: Type for block identifiers (e.g., `u64`, hash, slot).
- **`TxId`**: Type for transaction identifiers (e.g., hash, signature).
- **`Error`**: Adapter-specific error type, must implement `std::error::Error`.

---

## **Core Methods**

### 1. `latest_block`
Fetch the latest finalized block identifier.
- **Purpose:** For finality tracking, event polling, and health checks.
- **Returns:** `Result<BlockId, Error>`

### 2. `get_transaction`
Fetch transaction details by ID.
- **Purpose:** For status tracking, proof of inclusion, or replay.
- **Returns:** `Result<Option<Vec<u8>>, Error>`

### 3. `wait_for_finality`
Wait until a given block is finalized.
- **Purpose:** Ensures message or transaction is irreversible before proceeding.
- **Returns:** `Result<(), Error>`

### 4. `submit_message`
Submit a cross-chain message or proof to the chain.
- **Purpose:** Core relay operation; returns a transaction ID for tracking.
- **Returns:** `Result<TxId, Error>`

### 5. `listen_for_events`
Listen for new message events emitted by the chain.
- **Purpose:** Enables event-driven relaying and monitoring.
- **Returns:** `Result<Vec<MessageEvent>, Error>`

### 6. `verify_on_chain`
Optionally verify a message/proof on-chain (e.g., via contract call).
- **Purpose:** For ZK or signature verification, fraud proofs, etc.
- **Returns:** `Result<(), Error>`

### 7. `estimate_fee`
Estimate the native transaction fee for submitting a message.
- **Purpose:** For relayer economics, fee markets, and user feedback.
- **Returns:** `Result<u128, Error>`

### 8. `message_status`
Query the status of a message (pending, confirmed, failed, etc).
- **Purpose:** For pipeline progress, retries, and user feedback.
- **Returns:** `Result<MessageStatus, Error>`

### 9. `health_check`
Perform a health check (e.g., ping the node, check sync).
- **Purpose:** For monitoring, failover, and alerting.
- **Returns:** `Result<(), Error>`

---

## **Key Types**

### `FrostMessage`
Canonical cross-chain message structure, including payload, proof, and metadata.

### `MessageEvent`
Represents a message event emitted by the chain, including the message, transaction hash, and block number.

### `MessageStatus`
Enum for message pipeline state: `Pending`, `InFlight`, `Confirmed`, `Failed(String)`.

---

## **Implementing ChainAdapter**

To support a new chain:
1. **Extend `ChainId`** in the SDK.
2. **Implement `ChainAdapter`** for your chain, mapping each method to the chain’s API (RPC, SDK, etc).
3. **Handle errors robustly:** Use the provided `AdapterError` or your own error type.
4. **Test:** Use the trait’s async nature to write integration and unit tests.

**Example:**
```rust
pub struct MyChainAdapter { /* ... */ }

#[async_trait]
impl ChainAdapter for MyChainAdapter {
    type BlockId = u64;
    type TxId = [u8; 32];
    type Error = AdapterError;

    async fn latest_block(&self) -> Result<Self::BlockId, Self::Error> { /* ... */ }
    // ... implement other methods ...
}
```

---

## **Best Practices**

- **Async everywhere:** All methods are async for non-blocking I/O.
- **Error propagation:** Use rich error types for diagnostics and retries.
- **Extensible:** Add custom metadata or events as needed.
- **Mockable:** Use trait objects or test adapters for simulation.

---

## **FAQ**

**Q: Can I use ChainAdapter for non-EVM chains?**  
A: Yes! The trait is chain-agnostic. Implement it for Solana, Substrate, Cosmos, etc.

**Q: How do I handle chain-specific types?**  
A: Use the associated types (`BlockId`, `TxId`, `Error`) to match your chain’s primitives.

**Q: Is it safe for concurrent use?**  
A: Yes, all methods are `Send + Sync` and async-ready.

---

## **See Also**

- [`FrostMessage`](./frostmessage.md)
- [`ProofData`](./proofdata.md)
- [`Prover`](./prover.md)
- [`ZkPlug`](./zkplug.md)
- [Frostgate SDK API Docs](../src/lib.rs)

---

## **Contributing**

To add a new chain, open a PR with your `ChainAdapter` implementation and tests.  
For questions, see [CONTRIBUTING.md](../CONTRIBUTING.md) or join the Frostgate community!

---

**Frostgate: Secure, ZK-powered cross-chain messaging for the modular future.**