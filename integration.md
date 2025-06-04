# Frostgate Integration Guide (Subject to Review)

This guide walks through the process of integrating Frostgate into your application for cross-chain communication.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Basic Setup](#basic-setup)
4. [Chain Integration](#chain-integration)
5. [Message Handling](#message-handling)
6. [Advanced Features](#advanced-features)
7. [Troubleshooting](#troubleshooting)

## Prerequisites

- Rust 1.70 or later
- CMake 3.10 or later
- Protocol Buffers compiler
- Node.js 16+ (for web components)
- Access to blockchain node RPC endpoints

## Installation

1. Add dependencies to `Cargo.toml`:

```toml
[dependencies]
frostgate-sdk = { git = "https://github.com/frostgate/frostgate-sdk.git" }
frostgate-icap = { git = "https://github.com/frostgate/frostgate-icap.git" }
tokio = { version = "1", features = ["full"] }
```

2. Optional components:

```toml
[dependencies]
frostgate-prover = { git = "https://github.com/frostgate/frostgate-prover.git" }
frostgate-verifier = { git = "https://github.com/frostgate/frostgate-verifier.git" }
```

## Basic Setup

1. Initialize the SDK:

```rust
use frostgate_sdk::{FrostMessage, ChainId};
use frostgate_icap::EthereumAdapter;

// Configure the adapter
let config = EthereumConfig {
    rpc_url: "https://mainnet.infura.io/v3/YOUR-KEY",
    private_key: "your-private-key",
    confirmations: 12,
};

// Create adapter instance
let adapter = EthereumAdapter::new(config);
```

2. Create a message:

```rust
let message = FrostMessage::new(
    ChainId::Ethereum,
    ChainId::Polkadot,
    payload.to_vec(),
    nonce,
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs(),
);
```

3. Submit the message:

```rust
let tx_hash = adapter.submit_message(&message).await?;
adapter.wait_for_finality(&tx_hash).await?;
```

## Chain Integration

### Ethereum Integration

1. Create Ethereum adapter:

```rust
use frostgate_icap::ethereum::{EthereumAdapter, EthereumConfig};

let config = EthereumConfig {
    rpc_url: "https://mainnet.infura.io/v3/YOUR-KEY",
    private_key: "your-private-key",
    confirmations: 12,
};

let adapter = EthereumAdapter::new(config);
```

2. Deploy contracts:

```bash
# Deploy verifier contract
forge create --rpc-url $RPC_URL \
    --private-key $PRIVATE_KEY \
    contracts/FrostgateVerifier.sol:FrostgateVerifier
```

### Polkadot Integration

1. Create Polkadot adapter:

```rust
use frostgate_icap::substrate::{PolkadotAdapter, PolkadotConfig};

let config = PolkadotConfig::mainnet(
    "wss://rpc.polkadot.io",
    "your-seed-phrase",
);

let adapter = PolkadotAdapter::new(config).await?;
```

2. Install pallet:

Add to `runtime/Cargo.toml`:
```toml
pallet-frostgate = { git = "https://github.com/frostgate/frostgate.git" }
```

### Solana Integration

1. Create Solana adapter:

```rust
use frostgate_icap::solana::{SolanaAdapter, SolanaConfig};

let config = SolanaConfig {
    rpc_url: "https://api.mainnet-beta.solana.com",
    payer: payer_keypair,
    commitment: CommitmentLevel::Finalized,
};

let adapter = SolanaAdapter::new(config);
```

2. Deploy program:

```bash
solana program deploy target/deploy/frostgate_verifier.so
```

## Message Handling

### Sending Messages

1. Basic message:

```rust
let message = FrostMessage::new(
    ChainId::Ethereum,
    ChainId::Polkadot,
    payload,
    nonce,
    timestamp,
);

let tx_hash = adapter.submit_message(&message).await?;
```

2. With metadata:

```rust
let mut message = FrostMessage::new(/* ... */);
message.metadata.insert("version".to_string(), "1.0".to_string());
message.fee = Some(1_000_000); // Set fee in wei/planck
```

### Receiving Messages

1. Listen for events:

```rust
let events = adapter.listen_for_events().await?;

for event in events {
    match event.message.to_chain {
        ChainId::Ethereum => handle_ethereum_message(&event),
        ChainId::Polkadot => handle_polkadot_message(&event),
        ChainId::Solana => handle_solana_message(&event),
        _ => log::warn!("Unsupported chain"),
    }
}
```

2. Verify messages:

```rust
async fn handle_message(event: &MessageEvent) -> Result<(), Error> {
    // Verify on chain
    adapter.verify_on_chain(&event.message).await?;

    // Process payload
    process_payload(&event.message.payload).await?;

    Ok(())
}
```

## Advanced Features (planned)

### Batch Processing

1. Create batch processor:

```rust
use frostgate_sdk::BatchProcessor;

let processor = BatchProcessor::new(
    adapter.clone(),
    BatchConfig {
        max_size: 100,
        timeout: Duration::from_secs(60),
    },
);
```

2. Submit batch:

```rust
let messages = vec![message1, message2, message3];
processor.submit_batch(messages).await?;
```

### Custom Proof Generation

1. Implement prover:

```rust
use frostgate_prover::{Prover, ProverConfig};

let prover = Prover::new(
    ProverConfig {
        backend: "sp1",
        max_concurrent: 4,
    },
);

let proof = prover.generate_proof(program, input).await?;
```

2. Attach to message:

```rust
let mut message = FrostMessage::new(/* ... */);
message.proof = Some(proof);
```

## Troubleshooting

### Common Issues

1. **Connection Errors**
   - Check RPC endpoint availability
   - Verify network connectivity
   - Ensure correct chain ID

2. **Transaction Failures**
   - Check account balance
   - Verify gas/fee settings
   - Confirm nonce values

3. **Proof Verification**
   - Validate proving keys
   - Check circuit constraints
   - Verify input format

### Debugging

1. Enable logging:

```rust
use tracing_subscriber::{fmt, EnvFilter};

tracing_subscriber::fmt()
    .with_env_filter(EnvFilter::from_default_env())
    .init();
```

2. Monitor events:

```rust
use frostgate_sdk::monitoring::Monitor;

let monitor = Monitor::new(adapter.clone());
monitor.start().await?;
```

### Error Recovery

1. Implement retry logic:

```rust
use frostgate_sdk::retry::{RetryConfig, with_retry};

let config = RetryConfig {
    max_attempts: 3,
    backoff: Duration::from_secs(1),
};

let result = with_retry(config, || async {
    adapter.submit_message(&message).await
}).await?;
```

2. Handle failures:

```rust
match adapter.message_status(&message_id).await? {
    MessageStatus::Failed(error) => {
        log::error!("Message failed: {}", error);
        cleanup_failed_message(&message_id).await?;
    }
    _ => { /* ... */ }
}
```

## Best Practices

1. **Security**
   - Ensure to store private keys securely
   - Validate all input data
   - Monitor for suspicious activity

2. **Performance**
   - Use batch processing when possible
   - Implement proper caching
   - Monitor resource usage

3. **Reliability**
   - Implement proper error handling
   - Use retry mechanisms
   - Monitor system health

4. **Maintenance**
   - Keep dependencies updated
   - Monitor chain upgrades
   - Backup critical data

## Support

- Join our [Discord](https://discord.gg/frostgate)
- Check [Documentation](https://github.com/frostgate/frostgate-docs)
- Open [GitHub Issues](https://github.com/frostgate/frostgate/issues) 