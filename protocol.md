# Frostgate Protocol Specification

Version: 0.1.0
Status: Draft
Last Updated: 2024-02-14

## Table of Contents

1. [Introduction](#introduction)
2. [Message Format](#message-format)
3. [Chain Abstraction](#chain-abstraction)
4. [Zero-Knowledge Backend](#zero-knowledge-backend)
5. [Network Protocol](#network-protocol)
6. [Security Considerations](#security-considerations)

## Introduction

This document specifies the Frostgate protocol for trustless cross-chain communication. The protocol enables secure message passing between different blockchain networks using zero-knowledge proofs for validation across heterogenous chains.

## Message Format

### FrostMessage Structure

```rust
pub struct FrostMessage {
    pub id: Uuid,            // Unique message identifier
    pub from_chain: ChainId, // Source chain ID
    pub to_chain: ChainId,   // Destination chain ID
    pub payload: Vec<u8>,    // Application-specific data
    pub proof: Option<ZkProof>, // Zero-knowledge proof
    pub timestamp: u64,      // Message creation time
    pub nonce: u64,          // Replay protection
    pub signature: Option<Vec<u8>>, // Optional signature
    pub fee: Option<u128>,   // Protocol fee
    pub metadata: Option<HashMap<String, String>>, // Extended data
}
```

### Chain Identifiers

```rust
pub enum ChainId {
    Ethereum,    // ID: 0
    Polkadot,    // ID: 1
    Solana,      // ID: 2
    Unknown,     // Can be updated for future chains
}
```

### Message Events

```rust
pub struct MessageEvent {
    pub message: FrostMessage,
    pub tx_hash: Option<Vec<u8>>,
    pub block_number: Option<u64>,
}
```

## Chain Abstraction

### ChainAdapter Trait

```rust
#[async_trait]
pub trait ChainAdapter {
    type Error: std::error::Error;
    type TxId: Clone + Send + Sync;

    // Submit a message to the chain
    async fn submit_message(&self, msg: &FrostMessage) 
        -> Result<Self::TxId, Self::Error>;

    // Verify message exists on chain
    async fn verify_on_chain(&self, msg: &FrostMessage) 
        -> Result<(), Self::Error>;

    // Listen for relevant events
    async fn listen_for_events(&self) 
        -> Result<Vec<MessageEvent>, Self::Error>;

    // Get message status
    async fn message_status(&self, msg_id: &Uuid) 
        -> Result<MessageStatus, Self::Error>;

    // Wait for finality
    async fn wait_for_finality(&self, tx_id: &Self::TxId) 
        -> Result<(), Self::Error>;
}
```

### Implementation Requirements

1. **Message Submission**
   - Must validate message format
   - Must check nonce/replay protection
   - Must emit standardized events

2. **Event Handling**
   - Must filter relevant events
   - Must decode event data correctly
   - Must handle reorgs properly

3. **Finality**
   - Must respect chain-specific finality
   - Must handle temporary forks
   - Must verify sufficient confirmations

## Zero-Knowledge Backend

### ZkPlug Trait

```rust
#[async_trait]
pub trait ZkPlug: Send + Sync {
    // Execute program and generate proof
    async fn execute(&mut self, program: &[u8], input: &[u8]) 
        -> Result<ZkProof, ZkError>;

    // Verify a proof
    async fn verify(&self, proof: &ZkProof, input: Option<&[u8]>) 
        -> Result<bool, ZkError>;

    // Get proving/verifying keys
    async fn get_keys(&self) -> Result<(Vec<u8>, Vec<u8>), ZkError>;

    // Resource usage stats
    async fn get_resource_usage(&self) -> Result<ResourceUsage, ZkError>;
}
```

### Proof Requirements

1. **Completeness**
   - Valid proofs must always verify
   - All valid state transitions must be provable

2. **Soundness**
   - Invalid proofs must never verify
   - Must be secure against quantum attacks

3. **Zero-Knowledge**
   - Proofs must not reveal witness data
   - Must maintain privacy properties

## Network Protocol

### Message Flow

1. **Initialization**
   ```
   Application -> Source Chain
   - Create FrostMessage
   - Set nonce and timestamp
   - Submit for source chain
   ```

2. **Proof Generation**
   ```
   Prover -> ZK Backend
   - Observe chain event
   - Generate circuit inputs
   - Execute proof generation
   ```

3. **Message Relay**
   ```
   Relayer -> Destination Chain
   - Package message + proof
   - Submit to destination
   - Monitor confirmation
   ```

4. **Verification**
   ```
   Destination Chain
   - Verify proof validity
   - Check message format
   - Execute if valid
   ```

### Error Handling

1. **Network Errors**
   - Must implement exponential backoff
   - Must handle temporary failures
   - Must preserve message ordering

2. **Proof Errors**
   - Must detect invalid proofs
   - Must handle proving failures
   - Must support proof updates

3. **Chain Errors**
   - Must handle reorgs
   - Must retry failed transactions
   - Must track message status

## Security Considerations

### Threat Model

1. **Assumptions**
   - Source chain is honest
   - Source chain's finality assumption is solid
   - ZK system is secure (soundness of zk proofs)
   - Destination chain executes correctly

2. **Non-Assumptions**
   - Relayer honesty
   - Network reliability
   - Clock synchronization

### Attack Vectors

1. **Replay Attacks**
   - Prevented by nonce
   - Chain-specific tracking
   - Temporal constraints

2. **Front-running**
   - Batch processing
   - Fee mechanisms
   - Priority queues

3. **Denial of Service**
   - Multiple relayers
   - Rate limiting
   - Resource quotas

### Cryptographic Requirements

1. **Proof System**
   - Post-quantum security
   - Efficient verification
   - Transparent setup

2. **Hash Functions**
   - Collision resistant
   - Pre-image resistant
   - Length extension protected

## References

1. [SP1 zkVM Documentation](https://github.com/succinctlabs/sp1)
2. [Substrate Documentation](https://docs.substrate.io/)
3. [Zero-Knowledge Proofs: An Illustrated Primer](https://blog.cryptographyengineering.com/2014/11/27/zero-knowledge-proofs-illustrated-primer/)
4. [Chain Interoperability](https://polkadot.network/PolkaDotPaper.pdf)

## Appendix

### A. Message Serialization

```rust
impl Serialize for FrostMessage {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        // Implementation details
    }
}
```

### B. Circuit Specifications

```rust
pub struct CircuitInputs {
    pub block_header: Vec<u8>,
    pub state_proof: Vec<u8>,
    pub message_data: Vec<u8>,
}
```

### C. Version History

- 0.1.0: Initial protocol specification
- 0.0.1: Draft proposal 