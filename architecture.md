# Frostgate Architecture (Subject to Review)

This document describes the high-level architecture of Frostgate, a ZK-agnostic modular architecture for trustless blockchain interoperability.

## System Overview

Frostgate is designed as a modular system with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                     Application Layer                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────┼─────────────────────────────────┐
│                    Protocol Layer                            │
│   ┌─────────────┐    ┌────┴─────┐    ┌──────────────┐      │
│   │    ICAP     │◄──►│ FrostMsg │◄──►│    ZKPlug    │      │
│   └─────────────┘    └────┬─────┘    └──────────────┘      │
└───────────────────────────┼─────────────────────────────────┘
                            │
┌───────────────────────────┼─────────────────────────────────┐
│                   Implementation Layer                       │
│   ┌─────────────┐    ┌────┴─────┐    ┌──────────────┐      │
│   │   Chain     │    │  Proof   │    │     ZK       │      │
│   │  Adapters   │    │ Pipeline │    │   Backends   │      │
│   └─────────────┘    └────┬─────┘    └──────────────┘      │
└───────────────────────────┼─────────────────────────────────┘
                            │
┌───────────────────────────┼─────────────────────────────────┐
│                    Network Layer                             │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. ICAP (Interoperable Chain Abstraction Protocol)

ICAP provides a unified interface for interacting with different blockchain networks:

```rust
#[async_trait]
pub trait ChainAdapter {
    type Error: std::error::Error;
    type TxId: Clone + Send + Sync;

    async fn submit_message(&self, msg: &FrostMessage) -> Result<Self::TxId, Self::Error>;
    async fn verify_on_chain(&self, msg: &FrostMessage) -> Result<(), Self::Error>;
    async fn listen_for_events(&self) -> Result<Vec<MessageEvent>, Self::Error>;
    // ... other methods
}
```

### 2. ZKPlug (Zero-Knowledge Plugin System)

ZKPlug enables support for multiple zero-knowledge proof systems:

```rust
#[async_trait]
pub trait ZkPlug: Send + Sync {
    async fn execute(&mut self, program: &[u8], input: &[u8]) -> Result<ZkProof, ZkError>;
    async fn verify(&self, proof: &ZkProof, input: Option<&[u8]>) -> Result<bool, ZkError>;
    // ... other methods
}
```

### 3. Message Format

The `FrostMessage` structure defines the canonical cross-chain message format:

```rust
pub struct FrostMessage {
    pub id: Uuid,
    pub from_chain: ChainId,
    pub to_chain: ChainId,
    pub payload: Vec<u8>,
    pub proof: Option<ZkProof>,
    pub timestamp: u64,
    pub nonce: u64,
    // ... other fields
}
```

## Message Flow

1. **Message Creation**
   - Application creates a `FrostMessage`
   - Chain adapter validates message format
   - Message is submitted for the source chain

2. **Proof Generation**
   - Prover observes source chain events
   - Generates ZK proof of valid state transition
   - Proof is attached to message

3. **Message Relay**
   - Relayer picks up message + proof
   - Submits to destination chain
   - Handles retries and confirmations

4. **Verification**
   - Destination chain verifies ZK proof
   - If valid, executes message payload
   - Emits verification event

## Security Model

### Trust Assumptions

- Source chain consensus is honest
- ZK proof system is sound
- Destination chain executes correctly
- No assumptions about relayers

### Security Properties

1. **Authenticity**
   - Messages cannot be forged
   - Source chain events are cryptographically verified

2. **Non-repudiation**
   - All messages are permanently recorded
   - Proofs provide cryptographic evidence

3. **Atomicity**
   - Messages are executed exactly once
   - No partial message execution

4. **Censorship Resistance**
   - Any party can act as relayer
   - Multiple relayers can operate simultaneously

## Scalability Considerations

### Horizontal Scaling

- Independent proof generation
- Parallel message processing
- Multiple active relayers

### Vertical Scaling

- Batch proof generation
- Optimized verification circuits
- Efficient state management

## Implementation Details

### Chain Adapters

Currently implemented adapters:
- Ethereum (via ethers)
- Polkadot/Substrate (via subxt)
- Solana (via solana-client)
- Sui (planned)

### ZK Backends

Supported proving systems:
- SP1 zkVM (primary)
- RISC Zero (planned)
- Halo2 (planned)

### Network Protocol

Message relay protocol:
1. Subscribe to source chain events
2. Generate/verify proofs
3. Submit to destination chain
4. Monitor execution status

## Future Extensions

1. **Additional Features**
   - Multi-proof aggregation
   - Cross-chain asset transfers
   - Complex state transitions

2. **Planned Integrations**
   - More blockchain networks
   - Additional ZK backends
   - Layer 2 solutions

3. **Protocol Upgrades**
   - Enhanced privacy features
   - Improved scalability
   - Extended message capabilities

## References

- [Protocol Specification](./protocol.md)
- [API Documentation](./api.md)
- [Integration Guide](./integration.md)
- [Security Analysis](./security.md) 