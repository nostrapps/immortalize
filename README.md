# Immortalize

Anchor your Nostr identity to Bitcoin using [Blocktrails](https://blocktrails.org) technology.

## What it does

Immortalize creates a cryptographic proof that permanently links your Nostr profile or notes to a Bitcoin transaction. This proof:

- Uses P2TR (Taproot) key tweaking - no OP_RETURN data bloat
- Is verifiable by anyone with the TXO URI and your public key
- Survives even if Nostr relays go offline
- Works on Bitcoin mainnet, testnet4, and Litecoin

## How it works

1. **Connect** your Nostr identity (via extension or nsec)
2. **Fund** the app's faucet address (testnet4 sats)
3. **Immortalize** your profile or a note
4. **Share** the proof URI for others to verify

### Technical Flow

```
TX1: Faucet UTXO → Genesis UTXO (your pubkey) + Change
TX2: Genesis UTXO → Anchor UTXO (tweaked pubkey)

Where: tweaked_pubkey = your_pubkey + SHA256(event_hash) × G
```

The anchor output's pubkey encodes a commitment to your Nostr event data.

## Live Demo

**https://nostrapps.github.io/immortalize**

## Files

- `index.html` - Main application
- `verify.html` - Proof verification page
- `docs.html` - Documentation

## Self-Funding Faucet

The app generates a unique faucet address for each session. Fund it using any [testnet4 faucet](https://github.com/testnet4/awesome-testnet4#faucets), and you can create multiple proofs from that balance.

## Verification

Proofs can be verified by:
1. Fetching the anchor transaction from the blockchain
2. Computing the expected tweaked pubkey from (base_pubkey, event_hash)
3. Comparing against the actual anchor output pubkey

## Built With

- [Preact](https://preactjs.com/) + [HTM](https://github.com/developit/htm) - UI
- [@noble/curves](https://github.com/paulmillr/noble-curves) - Cryptography
- [@noble/hashes](https://github.com/paulmillr/noble-hashes) - Hashing
- [Blocktrails](https://blocktrails.org) - Anchoring protocol
- [Mempool.space API](https://mempool.space/docs/api) - Blockchain data

## License

MIT
