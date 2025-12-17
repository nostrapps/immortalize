# CLAUDE.md - Immortalize

## Project Overview

Immortalize is a client-side web app that anchors Nostr profiles and notes to Bitcoin using Blocktrails P2TR key tweaking technology.

## Architecture

**Single-file HTML apps** - Each page is self-contained with inline JS/CSS:
- `index.html` - Main app (~1800 lines)
- `verify.html` - Proof verification (~620 lines)
- `docs.html` - Documentation

**No build process** - Uses ES modules from esm.sh CDN:
```javascript
import { secp256k1, schnorr } from 'https://esm.sh/@noble/curves@1.4.0/secp256k1';
import { sha256 } from 'https://esm.sh/@noble/hashes@1.4.0/sha256';
```

**UI Framework**: Preact + HTM (no JSX compilation needed)

## Key Concepts

### Blocktrails Anchoring
```
tweak = SHA256(data_hash) mod n
tweaked_privkey = base_privkey + tweak
tweaked_pubkey = base_pubkey + tweak × G
```

The anchor transaction output uses the tweaked pubkey, cryptographically committing to the data.

### 2-TX Flow
1. **TX1**: Faucet UTXO → Genesis (user pubkey) + Refreshed faucet
2. **TX2**: Genesis → Anchor (tweaked pubkey)

### TXO URI Format
```
txo:<chain>:<txid>:<vout>
```
Chains: `btc`, `tbtc4`, `ltc`

## Important Functions

### index.html

- `buildImmortalizeTxs()` - Builds both transactions
- `serializeTransaction()` - Creates hex + computes txid (legacy serialization for txid)
- `signTaprootInput()` - BIP341 Taproot signing with Schnorr
- `tweakPubkey()` / `tweakPrivkey()` - Blocktrails key tweaking
- `createAndSignEvent()` - Creates Nostr events (kind 30078)
- `fetchRecentImmortalized()` - Fetches proof records from relays

### verify.html

- `deriveExpectedPubkey()` - Computes expected anchor from base + hash
- `fetchTx()` - Gets transaction from mempool API
- `computeEventHash()` - Canonical Nostr event hash

## Nostr Integration

**Kind 30078** (Parameterized Replaceable) for proof records:
- `d` tag: profile pubkey (profiles) or note ID (notes)
- `t` tags: `immortalize`, `blocktrails`
- `chain` tag: chain ID
- Content: JSON with txo, tx1, tx2, dataHash, basePubkey

**Relays**:
```javascript
const DEFAULT_RELAYS = [
  'wss://nos.lol',
  'wss://relay.primal.net',
  'wss://purplepag.es',
  'wss://relay.nostr.band'
];
```

## Common Tasks

### Adding a new chain
1. Add to chain selector in index.html
2. Add mempool API base URL in `getApiBase()`
3. Add explorer URL in `getExplorerUrl()` (both files)
4. Update bech32 HRP if needed

### Debugging transaction issues
- Check `serializeTransaction()` - txid must use legacy (no witness) serialization
- Verify sighash computation in `signTaprootInput()`
- Ensure amounts/fees are correct (DUST = 10000 for testnet4, 546 for mainnet)

### Fixing relay issues
- Update `DEFAULT_RELAYS` array in both index.html and verify.html
- Check WebSocket connection timeout (currently 5s)

## Testing

No automated tests. Manual testing flow:
1. Connect with Nostr extension or nsec
2. Fund faucet address (use testnet4 faucets)
3. Immortalize profile → verify proof
4. Immortalize note → verify proof
5. Check "Recently Immortalized" updates

## Security Notes

- Private keys handled in browser only, never transmitted
- nsec input stored in sessionStorage (cleared on tab close)
- Faucet key can be in URL query string for persistence
