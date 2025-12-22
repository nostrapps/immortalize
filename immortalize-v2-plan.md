# Immortalize v2: Project Plan

## Anchor Any Nostr Note to Bitcoin. Forever.

---

## Overview

**Product:** Immortalize — anchor Nostr notes and profiles to Bitcoin using Blocktrails, creating permanent, verifiable proof of existence.

**Core Technology:** Blocktrails — Nostr-native state anchoring via P2TR key tweaking

**Architecture:** Unified Blocktrails mechanism across all supported chains

---

## Table of Contents

1. [Product Vision](#product-vision)
2. [Technical Architecture](#technical-architecture)
3. [Core Features](#core-features)
4. [User Interface](#user-interface)
5. [API Specification](#api-specification)
6. [Database Schema](#database-schema)
7. [Development Phases](#development-phases)
8. [Go-to-Market](#go-to-market)
9. [Business Model](#business-model)
10. [Success Metrics](#success-metrics)

---

## Product Vision

### The Problem

```
Nostr notes exist on relays.
Relays can disappear.
Notes can be deleted.
There's no proof of "when" that's trustless.
```

### The Solution

```
Anchor note hashes to Bitcoin using Blocktrails.
One P2TR output = permanent proof.
Nostr keys control the trail directly.
SPV-verifiable. No trust required.
```

### Why Blocktrails

| Approach | Problem |
|----------|---------|
| OP_RETURN | Data embedding, not native |
| Separate timestamps | Disconnected from identity |
| Centralized services | Trust required |
| **Blocktrails** | Nostr-native keys, P2TR, chainable state |

Your Nostr identity IS your Blocktrail. Same keys. No conversion.

---

## Technical Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        IMMORTALIZE                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐           │
│  │  Web App  │───▶│    API    │───▶│  Worker   │           │
│  │  (React)  │    │  (Node)   │    │  (Queue)  │           │
│  └───────────┘    └─────┬─────┘    └─────┬─────┘           │
│                         │                │                  │
│                         ▼                ▼                  │
│                   ┌───────────┐    ┌───────────┐           │
│                   │ Postgres  │    │Blocktrails│           │
│                   │           │    │  Library  │           │
│                   └───────────┘    └─────┬─────┘           │
│                                          │                  │
│                         ┌────────────────┼────────────────┐ │
│                         ▼                ▼                ▼ │
│                   ┌──────────┐    ┌──────────┐    ┌──────┐ │
│                   │ Bitcoin  │    │ Testnet4 │    │ LTC  │ │
│                   │ Mainnet  │    │  (dev)   │    │(free)│ │
│                   └──────────┘    └──────────┘    └──────┘ │
│                                                             │
│                   ┌───────────┐    ┌───────────┐           │
│                   │   Nostr   │    │ Lightning │           │
│                   │  Relays   │    │   (LND)   │           │
│                   └───────────┘    └───────────┘           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Chain Configuration

```typescript
type Chain = 'btc' | 'testnet4' | 'ltc';

const chains: Record<Chain, ChainConfig> = {
  btc: {
    name: 'Bitcoin',
    network: bitcoin.networks.bitcoin,
    addressPrefix: 'bc1p',
    explorer: 'mempool.space',
    electrum: 'electrum.blockstream.info:50002',
    enabled: true,
    default: true
  },
  testnet4: {
    name: 'Testnet4',
    network: bitcoin.networks.testnet,
    addressPrefix: 'tb1p',
    explorer: 'mempool.space/testnet4',
    electrum: 'electrum.testnet4.io:50002',
    enabled: true,
    default: false
  },
  ltc: {
    name: 'Litecoin',
    network: litecoin.networks.mainnet,
    addressPrefix: 'ltc1p',
    explorer: 'blockchair.com/litecoin',
    electrum: 'electrum-ltc.bysh.me:50002',
    enabled: true,
    default: false
  }
};
```

### TXO URI Scheme

```
txo:<chain>:<p2tr_address>/<block_height>

Examples:
  txo:btc:bc1pqyqszqgpqyqszqgpqyqszqgpqyqs.../850000
  txo:testnet4:tb1pxyz.../200000
  txo:ltc:ltc1pabc.../2800000
```

### Blocktrails Integration

```typescript
import { Blocktrail } from 'blocktrails';

interface ImmortalizationState {
  v: 1;                    // version
  n: string;               // note id (hex)
  h: string;               // note hash (sha256)
  p: string;               // author pubkey (hex)
  t: number;               // note created_at
  a: number;               // anchor timestamp
}

class ImmortalizationTrail {
  private trail: Blocktrail;
  private chain: Chain;

  constructor(nostrPrivateKey: string, chain: Chain = 'btc') {
    this.chain = chain;
    this.trail = new Blocktrail(nostrPrivateKey, {
      network: chains[chain].network
    });
  }

  async anchor(note: NostrEvent): Promise<TxoUri> {
    const state: ImmortalizationState = {
      v: 1,
      n: note.id,
      h: this.computeHash(note),
      p: note.pubkey,
      t: note.created_at,
      a: Math.floor(Date.now() / 1000)
    };

    const output = this.trail.advance(JSON.stringify(state));
    const txid = await this.broadcast(output);
    const block = await this.waitForConfirmation(txid);

    return `txo:${this.chain}:${output.p2trAddress}/${block}`;
  }

  private computeHash(note: NostrEvent): string {
    // NIP-01 canonical serialization
    const canonical = JSON.stringify([
      0,
      note.pubkey,
      note.created_at,
      note.kind,
      note.tags,
      note.content
    ]);
    return sha256(canonical);
  }
}
```

### Verification Flow

```typescript
async function verify(noteId: string, txoUri: string): Promise<VerificationResult> {
  // 1. Parse TXO URI
  const { chain, address, blockHeight } = parseTxoUri(txoUri);

  // 2. Fetch note from relays
  const note = await fetchNote(noteId);
  if (!note) {
    return { valid: false, error: 'note_not_found' };
  }

  // 3. Compute expected hash
  const expectedHash = computeNoteHash(note);

  // 4. Fetch transaction spending to this address
  const tx = await fetchTxByAddress(chain, address);
  if (!tx) {
    return { valid: false, error: 'tx_not_found' };
  }

  // 5. Verify block height
  if (tx.blockHeight !== blockHeight) {
    return { valid: false, error: 'block_mismatch' };
  }

  // 6. Fetch state from trail storage
  const state = await fetchTrailState(txoUri);
  if (!state) {
    return { valid: false, error: 'state_not_found' };
  }

  // 7. Verify hash matches
  if (state.h !== expectedHash) {
    return { valid: false, error: 'hash_mismatch' };
  }

  // 8. Verify Blocktrails chain (key tweaking)
  const trailValid = await verifyTrailChain(state, address);
  if (!trailValid) {
    return { valid: false, error: 'trail_invalid' };
  }

  return {
    valid: true,
    chain,
    blockHeight,
    timestamp: tx.blockTime,
    noteHash: expectedHash,
    author: note.pubkey
  };
}
```

---

## Core Features

### 1. Note Immortalization

```
Input:  Nostr note identifier
        - nevent1...
        - note1...
        - nostr: URI
        - Relay URL with note

Output: TXO URI proof
        - txo:btc:bc1p.../850000

Flow:
1. Parse note identifier → extract note ID
2. Fetch note from multiple relays
3. Validate note exists and is signed
4. Compute canonical hash
5. Create Blocktrails state
6. Generate P2TR output
7. Broadcast transaction
8. Wait for confirmation
9. Return TXO URI + proof bundle
```

### 2. Profile Immortalization

```
Input:  Nostr pubkey
        - npub1...
        - hex pubkey

Output: TXO URI proof of kind:0 profile

Flow:
1. Fetch latest kind:0 for pubkey
2. Hash profile content
3. Anchor via Blocktrails
4. Return proof
```

### 3. Thread Immortalization

```
Input:  Root note ID
        - Fetches all replies in thread

Output: Single TXO URI anchoring thread merkle root

Flow:
1. Fetch root note
2. Fetch all reply notes (kind:1 with e-tag)
3. Build merkle tree of note hashes
4. Anchor merkle root
5. Return proof + individual note positions
```

### 4. Verification

```
Input:  Note ID + TXO URI

Output: Verification result
        - Valid/Invalid
        - Chain details
        - Block timestamp
        - Explorer links

Verification levels:
1. Quick: Check TXO exists
2. Standard: Verify hash matches
3. Full: Verify complete trail chain
```

### 5. Proof Export

```
Formats:
- JSON proof bundle
- QR code (TXO URI)
- Nostr event (kind:10043)
- HTML certificate
```

---

## User Interface

### Home / Immortalize

```
┌─────────────────────────────────────────────────────────────┐
│  ⛓️ IMMORTALIZE                           [Connect Wallet]  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│              Anchor any Nostr note to Bitcoin.              │
│                         Forever.                            │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  Paste note ID, nevent, or Nostr URL:              │   │
│  │                                                     │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │ nevent1qqsq3...                             │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │                                                     │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │                                             │   │   │
│  │  │  "Just mass adopted a mass adoption         │   │   │
│  │  │   of mass adoption" - @jack                 │   │   │
│  │  │                                             │   │   │
│  │  │  Kind: 1 (note)   Created: 2 hours ago     │   │   │
│  │  │                                             │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │                                                     │   │
│  │            ┌───────────────────────────┐           │   │
│  │            │  ⚡ Immortalize (2k sats) │           │   │
│  │            └───────────────────────────┘           │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ──────────────────────────────────────────────────────────│
│                                                             │
│  Recent Immortalizations                                    │
│                                                             │
│  ⛓️ note1abc... → btc #850,100           2 min ago         │
│  ⛓️ note1def... → btc #850,050           15 min ago        │
│  ⛓️ npub1xyz... → btc #850,000           1 hour ago        │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  Verify    Docs    API    GitHub              Powered by ⚡ │
└─────────────────────────────────────────────────────────────┘
```

### Advanced Options (Collapsed)

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ▼ Advanced Options                                         │
│                                                             │
│    Chain:                                                   │
│    ◉ Bitcoin mainnet (default)                             │
│    ○ Testnet4 (free, for testing)                          │
│    ○ Litecoin (free tier)                                  │
│                                                             │
│    ☐ Publish proof to Nostr (kind:10043)                   │
│    ☐ Include in public feed                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Verification Page

```
┌─────────────────────────────────────────────────────────────┐
│  ⛓️ IMMORTALIZE                                    Verify   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │              ✓ VERIFIED IMMORTALIZATION             │   │
│  │                                                     │   │
│  │  ─────────────────────────────────────────────────  │   │
│  │                                                     │   │
│  │  Note                                               │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │ "Bitcoin is the exit."                      │   │   │
│  │  │                                             │   │   │
│  │  │ @melvin · 2 hours ago                       │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │                                                     │   │
│  │  Proof Details                                      │   │
│  │  ├─ Chain:      Bitcoin mainnet                    │   │
│  │  ├─ Block:      850,100                            │   │
│  │  ├─ Timestamp:  Jan 15, 2025 14:32:00 UTC          │   │
│  │  ├─ Address:    bc1pqyqs...7w4k                    │   │
│  │  ├─ TX:         a1b2c3d4...                        │   │
│  │  └─ Hash:       sha256:f8e9d0c1...                 │   │
│  │                                                     │   │
│  │  TXO URI                                            │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │ txo:btc:bc1pqyqs...7w4k/850100              │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │                                                     │   │
│  │  [View on Mempool] [View on Nostr] [Export Proof]  │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Payment Flow

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    ⚡ Pay 2,000 sats                        │
│                                                             │
│           ┌─────────────────────────────┐                  │
│           │                             │                  │
│           │         [QR CODE]           │                  │
│           │                             │                  │
│           │    lnbc20u1pjxyz...         │                  │
│           │                             │                  │
│           └─────────────────────────────┘                  │
│                                                             │
│                   [Open in Wallet]                         │
│                                                             │
│           WebLN detected: Alby                             │
│           [Pay with Alby]                                  │
│                                                             │
│  ──────────────────────────────────────────────────────────│
│                                                             │
│           Waiting for payment...  ◌                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Success State

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    ✓ IMMORTALIZED                          │
│                                                             │
│  Your note is now permanently anchored to Bitcoin.         │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  TXO URI:                                           │   │
│  │  txo:btc:bc1pqyqszqgpqyqs.../850100                │   │
│  │                                                 📋  │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Status: Confirming... (0/1 confirmations)                 │
│  Estimated: ~10 minutes                                    │
│                                                             │
│  [View Transaction]  [Share Proof]  [Immortalize Another] │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## API Specification

### Endpoints

#### POST /api/v1/immortalize

Create new immortalization.

```typescript
// Request
{
  "note_id": "nevent1..." | "note1..." | "<hex>",
  "chain": "btc" | "testnet4" | "ltc",  // default: "btc"
  "payment_hash": "<lightning payment hash>"
}

// Response
{
  "id": "uuid",
  "status": "pending" | "broadcasting" | "confirming" | "confirmed",
  "txo_uri": "txo:btc:bc1p.../850100",
  "txid": "a1b2c3...",
  "block": 850100,
  "confirmations": 0,
  "note": {
    "id": "...",
    "content": "...",
    "author": "npub1...",
    "created_at": 1705312320
  },
  "hash": "sha256:f8e9d0c1...",
  "proof_url": "https://immortalize.app/proof/uuid",
  "explorer_url": "https://mempool.space/tx/a1b2c3..."
}
```

#### GET /api/v1/verify

Verify an immortalization.

```typescript
// Request
GET /api/v1/verify?note_id=note1...&txo=txo:btc:bc1p.../850100

// Response
{
  "valid": true,
  "chain": "btc",
  "block": 850100,
  "block_timestamp": 1705312320,
  "confirmations": 6,
  "note_hash": "sha256:f8e9d0c1...",
  "computed_hash": "sha256:f8e9d0c1...",
  "hash_match": true,
  "trail_valid": true,
  "explorer_url": "https://mempool.space/tx/..."
}
```

#### GET /api/v1/status/:id

Check immortalization status.

```typescript
// Response
{
  "id": "uuid",
  "status": "confirmed",
  "txo_uri": "txo:btc:bc1p.../850100",
  "confirmations": 3,
  "estimated_confirmation": null
}
```

#### GET /api/v1/note/:note_id

Get immortalizations for a note.

```typescript
// Response
{
  "note_id": "...",
  "immortalizations": [
    {
      "txo_uri": "txo:btc:bc1p.../850100",
      "chain": "btc",
      "block": 850100,
      "timestamp": 1705312320
    }
  ]
}
```

#### POST /api/v1/invoice

Generate Lightning invoice.

```typescript
// Request
{
  "chain": "btc",
  "note_id": "nevent1..."
}

// Response
{
  "payment_request": "lnbc20u1...",
  "payment_hash": "abc123...",
  "amount_sats": 2000,
  "expires_at": 1705312320
}
```

### WebSocket

Real-time status updates.

```typescript
// Connect
ws://immortalize.app/ws

// Subscribe
{ "type": "subscribe", "id": "uuid" }

// Updates
{ "type": "status", "id": "uuid", "status": "confirmed", "confirmations": 1 }
```

---

## Database Schema

```sql
-- Chains enum
CREATE TYPE chain_type AS ENUM ('btc', 'testnet4', 'ltc');

-- Immortalization status
CREATE TYPE immortalization_status AS ENUM (
  'pending',      -- Payment received, not yet broadcast
  'broadcasting', -- Transaction being broadcast
  'confirming',   -- In mempool, waiting for block
  'confirmed',    -- At least 1 confirmation
  'failed'        -- Something went wrong
);

-- Core immortalizations table
CREATE TABLE immortalizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Note data
  note_id VARCHAR(64) NOT NULL,
  note_hash VARCHAR(64) NOT NULL,
  author_pubkey VARCHAR(64) NOT NULL,
  note_content TEXT,
  note_created_at TIMESTAMP,

  -- Chain data
  chain chain_type NOT NULL DEFAULT 'btc',
  txid VARCHAR(64),
  p2tr_address VARCHAR(128),
  block_height INTEGER,
  block_timestamp TIMESTAMP,

  -- Blocktrails data
  trail_state JSONB NOT NULL,
  state_index INTEGER NOT NULL DEFAULT 0,

  -- Status
  status immortalization_status NOT NULL DEFAULT 'pending',
  confirmations INTEGER DEFAULT 0,

  -- Timestamps
  created_at TIMESTAMP DEFAULT NOW(),
  broadcast_at TIMESTAMP,
  confirmed_at TIMESTAMP,

  -- Indexes
  CONSTRAINT unique_note_chain UNIQUE (note_id, chain)
);

CREATE INDEX idx_note_id ON immortalizations(note_id);
CREATE INDEX idx_txid ON immortalizations(txid);
CREATE INDEX idx_status ON immortalizations(status);
CREATE INDEX idx_chain ON immortalizations(chain);
CREATE INDEX idx_author ON immortalizations(author_pubkey);
CREATE INDEX idx_created ON immortalizations(created_at DESC);

-- Payments
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  immortalization_id UUID REFERENCES immortalizations(id),

  payment_hash VARCHAR(64) NOT NULL UNIQUE,
  payment_request TEXT NOT NULL,
  amount_sats INTEGER NOT NULL,

  status VARCHAR(20) NOT NULL DEFAULT 'pending',

  created_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP NOT NULL,
  paid_at TIMESTAMP
);

CREATE INDEX idx_payment_hash ON payments(payment_hash);
CREATE INDEX idx_payment_status ON payments(status);

-- Trail states (for verification)
CREATE TABLE trail_states (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  txo_uri VARCHAR(256) NOT NULL UNIQUE,

  chain chain_type NOT NULL,
  p2tr_address VARCHAR(128) NOT NULL,
  block_height INTEGER NOT NULL,

  state_json JSONB NOT NULL,
  previous_txo_uri VARCHAR(256),

  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_txo_uri ON trail_states(txo_uri);
CREATE INDEX idx_trail_address ON trail_states(p2tr_address);

-- Public feed (opt-in)
CREATE TABLE public_feed (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  immortalization_id UUID REFERENCES immortalizations(id),

  displayed_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_feed_time ON public_feed(displayed_at DESC);
```

---

## Development Phases

### Phase 1: Foundation (Week 1-2)

#### Week 1: Infrastructure

```
Tasks:
├─ [ ] Initialize monorepo (turborepo or nx)
│     ├─ packages/web (React + Vite + TypeScript)
│     ├─ packages/api (Node + Express + TypeScript)
│     ├─ packages/worker (Bull queue)
│     └─ packages/shared (types, utils)
├─ [ ] Set up PostgreSQL (Railway or Supabase)
├─ [ ] Configure CI/CD (GitHub Actions)
├─ [ ] Set up Electrum connections
│     ├─ Bitcoin mainnet
│     ├─ Testnet4
│     └─ Litecoin mainnet
├─ [ ] Integrate Blocktrails library
└─ [ ] Basic health check endpoints

Deliverables:
├─ Repo structure complete
├─ Database deployed
├─ Can connect to all 3 chains
└─ CI pipeline green
```

#### Week 2: Nostr + Blocktrails Core

```
Tasks:
├─ [ ] Nostr note fetching
│     ├─ Parse nevent, note1, hex IDs
│     ├─ Multi-relay fallback
│     └─ Validate signatures
├─ [ ] Note hash computation (NIP-01 canonical)
├─ [ ] Blocktrails integration
│     ├─ Trail creation
│     ├─ State advancement
│     └─ P2TR output generation
├─ [ ] Transaction broadcasting
│     ├─ Bitcoin mainnet
│     ├─ Testnet4
│     └─ Litecoin
├─ [ ] Confirmation monitoring
└─ [ ] TXO URI generation

Deliverables:
├─ Can fetch any Nostr note
├─ Can anchor to testnet4
├─ End-to-end flow works (no payments)
└─ Verification logic complete
```

### Phase 2: Payments + UI (Week 3-4)

#### Week 3: Lightning + Queue

```
Tasks:
├─ [ ] LND/CLN connection (or Voltage/Alby)
├─ [ ] Invoice generation
├─ [ ] Payment verification
├─ [ ] Job queue for immortalizations
│     ├─ Payment received → queue job
│     ├─ Broadcast transaction
│     ├─ Monitor confirmations
│     └─ Update status
├─ [ ] WebSocket for real-time updates
└─ [ ] Failure handling + refunds

Deliverables:
├─ Lightning invoices working
├─ Payment → immortalization flow
├─ Real-time status updates
└─ Error recovery working
```

#### Week 4: Frontend

```
Tasks:
├─ [ ] Landing page
├─ [ ] Note input + preview
├─ [ ] Chain selector (advanced options)
├─ [ ] Lightning payment modal
│     ├─ QR code display
│     ├─ WebLN integration
│     └─ Payment status
├─ [ ] Success state
├─ [ ] Verification page
├─ [ ] Recent immortalizations feed
├─ [ ] Mobile responsive
└─ [ ] Dark mode

Deliverables:
├─ Complete web app
├─ Payment flow working
├─ Verification working
└─ Mobile-friendly
```

### Phase 3: Polish + Launch (Week 5-6)

#### Week 5: Hardening

```
Tasks:
├─ [ ] Error handling audit
├─ [ ] Input validation (security)
├─ [ ] Rate limiting
├─ [ ] Loading states + skeletons
├─ [ ] Edge case testing
│     ├─ Invalid note IDs
│     ├─ Deleted notes
│     ├─ Payment timeouts
│     └─ Chain reorgs
├─ [ ] Performance optimization
├─ [ ] Logging + monitoring (Sentry, Axiom)
└─ [ ] Documentation

Deliverables:
├─ Production-ready
├─ All errors handled gracefully
├─ Sub-2s response times
└─ Monitoring in place
```

#### Week 6: Soft Launch

```
Tasks:
├─ [ ] Deploy to production
├─ [ ] SSL + domain configuration
├─ [ ] Fund Lightning node
├─ [ ] Fund chain wallets (small amounts)
├─ [ ] Invite 50 beta testers
├─ [ ] Monitor and fix issues
├─ [ ] Gather feedback
└─ [ ] Legal pages (ToS, Privacy)

Deliverables:
├─ Live on production
├─ Beta users onboarded
├─ First real immortalizations
└─ Feedback collected
```

### Phase 4: Public Launch (Week 7-8)

#### Week 7: Launch

```
Tasks:
├─ [ ] Public announcement
│     ├─ Nostr (personal + @immortalize)
│     ├─ Twitter/X
│     └─ Relevant communities
├─ [ ] Demo content
├─ [ ] First 100 immortalizations free (promo)
├─ [ ] Press/influencer outreach
├─ [ ] Monitor scaling
└─ [ ] Rapid bug fixes

Deliverables:
├─ Public launch complete
├─ Media coverage in Nostr community
├─ 500+ immortalizations
└─ Stable under load
```

#### Week 8: Integrations

```
Tasks:
├─ [ ] NIP-XX draft for immortalization events
├─ [ ] JavaScript SDK
├─ [ ] Reach out to client devs
│     ├─ Damus
│     ├─ Amethyst
│     ├─ Primal
│     └─ Snort
├─ [ ] Developer documentation
├─ [ ] Example integration code
└─ [ ] API keys for partners

Deliverables:
├─ NIP draft published
├─ SDK on npm
├─ At least 1 client integration started
└─ Dev docs complete
```

---

## Go-to-Market

### Target Users

| Segment | Who | Why They Care | Volume |
|---------|-----|---------------|--------|
| Power users | 1k+ followers | Preserve reputation | 10-50k |
| Influencers | 10k+ followers | Prove predictions | 1-5k |
| Developers | Client/tool builders | Integrate feature | 100-500 |
| Archivists | History preservers | Document events | 1-5k |

### Messaging

**Primary:**
> Anchor any Nostr note to Bitcoin. Forever.

**Supporting:**
> Your note. Your keys. Your proof.
> Relays forget. Bitcoin doesn't.
> Immortalize your predictions. Prove you called it.

**Technical:**
> Powered by Blocktrails — Nostr-native state on Bitcoin.

### Launch Channels

| Channel | Effort | Impact | Timing |
|---------|--------|--------|--------|
| Nostr native | High | Primary | Launch week |
| Twitter/X | Medium | Awareness | Launch week |
| Developer outreach | High | Multiplier | Week 8+ |
| Podcasts | Low | Credibility | Month 2+ |
| Content/SEO | Low | Long-term | Ongoing |

---

## Business Model

### Pricing

| Product | Chain | Price | Notes |
|---------|-------|-------|-------|
| Single note | BTC | 2000 sats | Default, premium |
| Single note | Testnet4 | Free | Testing only |
| Single note | LTC | Free | Budget tier |
| Profile | BTC | 3000 sats | kind:0 anchor |
| Thread (≤10) | BTC | 10000 sats | Merkle root |
| Bulk 100 | BTC | 150000 sats | 25% discount |
| API call | BTC | 1500 sats | Developer tier |

### Cost Structure

| Item | Monthly | Notes |
|------|---------|-------|
| Hosting (Railway) | $50 | API + worker + DB |
| Frontend (Vercel) | $0 | Free tier |
| Lightning (Voltage) | $30 | Or self-hosted |
| Electrum servers | $0 | Public or self-hosted |
| Domain + DNS | $5 | Cloudflare |
| Monitoring | $0-20 | Free tiers available |
| **Total fixed** | **~$100** | Before tx costs |

| Transaction costs | Per unit |
|-------------------|----------|
| BTC tx fee | ~$0.10-0.50 |
| LTC tx fee | ~$0.001 |
| Testnet4 | Free |

### Unit Economics

```
BTC immortalization:
  Revenue:     2000 sats (~$0.60)
  TX cost:     ~500 sats (~$0.15)
  Margin:      ~1500 sats (~$0.45)

LTC immortalization:
  Revenue:     0 (free tier)
  TX cost:     ~5 sats (~$0.001)
  Margin:      -5 sats (loss leader)
```

### Projections

```
Month 1:   1,000 BTC immortalizations × $0.45 = $450
Month 3:   5,000 × $0.45 = $2,250
Month 6:   15,000 × $0.45 = $6,750
Month 12:  50,000 × $0.45 = $22,500/month

Break-even: ~250 immortalizations/month
Target: 5,000+/month by month 6
```

---

## Success Metrics

### North Star

**Total Immortalizations (cumulative)**

| Milestone | Target | Timeframe |
|-----------|--------|-----------|
| First 1,000 | Week 8 | Launch |
| 10,000 | Month 3 | Growth |
| 50,000 | Month 6 | Traction |
| 200,000 | Month 12 | Scale |

### Supporting Metrics

| Category | Metric | M1 | M6 | M12 |
|----------|--------|----|----|-----|
| Acquisition | Unique users | 500 | 5k | 20k |
| Activation | Visitor → immortalize | 15% | 20% | 25% |
| Retention | Users 2+ immortalizations | 30% | 40% | 50% |
| Revenue | MRR | $500 | $5k | $20k |
| Technical | Uptime | 99% | 99.5% | 99.9% |
| Technical | Avg response time | <1s | <500ms | <300ms |

---

## Tech Stack Summary

### Frontend
- React 18 + TypeScript
- Vite
- TailwindCSS
- @tanstack/react-query
- nostr-tools
- @getalby/lightning-tools

### Backend
- Node.js + TypeScript
- Express
- Prisma (PostgreSQL)
- Bull (Redis queue)
- blocktrails
- nostr-tools

### Infrastructure
- Railway (API + worker + DB)
- Vercel (frontend)
- Cloudflare (DNS + CDN)
- Voltage or self-hosted (LND)

### Chains
- Bitcoin mainnet (Electrum)
- Bitcoin testnet4 (Electrum)
- Litecoin mainnet (Electrum)

---

## NIP-XX Draft: Immortalization Events

```
NIP-XX: Blockchain Immortalization

`draft` `optional`

This NIP defines events for recording blockchain-anchored proofs
of Nostr events using the Blocktrails protocol.

## Event Kind

Kind `10043` is used for immortalization records.

## Tags

Required:
- `e` - Event ID being immortalized
- `p` - Pubkey of original author
- `txo` - TXO URI (txo:<chain>:<address>/<block>)
- `hash` - SHA256 hash of canonical event

Optional:
- `proof` - URL to full proof bundle

## Content

Empty string.

## Example

{
  "kind": 10043,
  "pubkey": "<immortalizer service pubkey>",
  "created_at": 1705312320,
  "tags": [
    ["e", "<immortalized note id>", "<relay>", "immortalized"],
    ["p", "<author pubkey>"],
    ["txo", "txo:btc:bc1pqyqs.../850100"],
    ["hash", "sha256:f8e9d0c1b2a3..."]
  ],
  "content": "",
  "sig": "..."
}

## Verification

Clients SHOULD verify by:
1. Fetching the original event
2. Computing canonical hash
3. Verifying TXO exists and contains commitment
4. Optionally verifying full Blocktrails chain

## Display

Clients MAY display ⛓️ indicator on immortalized events.
```

---

## Appendix: Environment Variables

```bash
# Database
DATABASE_URL=postgresql://...

# Redis (for queue)
REDIS_URL=redis://...

# Lightning
LND_REST_HOST=...
LND_MACAROON=...
# Or
ALBY_ACCESS_TOKEN=...

# Electrum servers
ELECTRUM_BTC=electrum.blockstream.info:50002
ELECTRUM_TESTNET4=electrum.testnet4.io:50002
ELECTRUM_LTC=electrum-ltc.bysh.me:50002

# Service keys
SERVICE_PRIVATE_KEY=... # For signing proofs

# Feature flags
ENABLE_TESTNET4=true
ENABLE_LTC=true
ENABLE_PUBLIC_FEED=true

# Monitoring
SENTRY_DSN=...
```

---

## Appendix: Local Development

```bash
# Clone
git clone https://github.com/blocktrails/immortalize
cd immortalize

# Install
pnpm install

# Set up env
cp .env.example .env.local
# Edit .env.local with your values

# Database
pnpm db:push
pnpm db:seed

# Run
pnpm dev

# Access
# Frontend: http://localhost:5173
# API: http://localhost:3000
# Queue dashboard: http://localhost:3000/admin/queues
```

---

*Immortalize v2 — Anchor any Nostr note to Bitcoin. Forever.*

*Built on Blocktrails — Nostr-native state on Bitcoin.*
