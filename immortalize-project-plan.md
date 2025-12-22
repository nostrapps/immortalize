# Immortalize: Project Plan

## Make Any Nostr Note Permanent. Forever.

---

## Executive Summary

**Product:** Immortalize — a service to anchor Nostr notes and profiles to blockchain, creating permanent, verifiable proof of existence and publication.

**First App in the Blocktrails Ecosystem**

**Timeline:** 12 weeks to full launch

**Budget:** $15-25K (bootstrappable)

**Revenue Target:** $50K ARR within 6 months

**Strategic Value:** Gateway to full Blocktrails adoption

---

## Table of Contents

1. [Vision & Strategy](#vision--strategy)
2. [Product Specification](#product-specification)
3. [Technical Architecture](#technical-architecture)
4. [Development Phases](#development-phases)
5. [Go-to-Market](#go-to-market)
6. [Business Model](#business-model)
7. [Success Metrics](#success-metrics)
8. [Risk Assessment](#risk-assessment)
9. [Resource Requirements](#resource-requirements)
10. [Timeline](#timeline)

---

## Vision & Strategy

### The Opportunity

```
Nostr users: 500K-1M active
Problem: Notes can be deleted, relays can disappear
Solution: Permanent blockchain anchoring
Market: Underserved, crypto-native, willing to pay
```

### Strategic Position

```
Immortalize (simple)
     │
     ▼
Evidence trails (medium)
     │
     ▼
Identity trails (Blocktrails full)
     │
     ▼
Agent infrastructure (platform)
```

**Immortalize is the gateway drug to full sovereignty.**

### Why Now

- Nostr growing rapidly
- Users understand "permanent" value
- Lightning payments mature
- LTC fees near-zero
- No competition in this exact space

---

## Product Specification

### Core Features (MVP)

#### 1. Note Immortalization

```
Input:  Nostr note ID (nevent, note, or URL)
Output: Blockchain anchor proof

Process:
1. Fetch note from relays
2. Compute hash (sha256 of canonical note)
3. Create LTC transaction with commitment
4. Wait for confirmation
5. Return proof (txid, block, timestamp)
```

#### 2. Profile Immortalization

```
Input:  Nostr pubkey (npub or hex)
Output: Blockchain anchor of kind:0 profile

Process:
1. Fetch latest kind:0 for pubkey
2. Compute hash of profile content
3. Create LTC transaction
4. Return proof
```

#### 3. Proof Verification

```
Input:  Note ID + claimed proof
Output: Valid/invalid + details

Process:
1. Fetch note from relays
2. Recompute hash
3. Verify hash exists in claimed transaction
4. Verify transaction in blockchain
5. Return verification result
```

### User Interface

#### Web App (Primary)

```
┌─────────────────────────────────────────────────────────────────┐
│  ⛓️ IMMORTALIZE                              [Connect Wallet]   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                 Make any Nostr note permanent.                  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │  Note ID or URL:                                        │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │ nevent1...                                      │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  │                                                         │    │
│  │  ○ Litecoin ($0.10)    ○ Bitcoin ($0.50)               │    │
│  │                                                         │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │           ⚡ Immortalize (500 sats)              │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  Recent Immortalizations                            [View All]  │
│                                                                 │
│  ⛓️ note1abc... → LTC #2,700,100         2 minutes ago         │
│  ⛓️ note1def... → LTC #2,700,050         18 minutes ago        │
│  ⛓️ npub1xyz... → LTC #2,700,000         1 hour ago            │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Verify a Note    About    API    GitHub           Powered by ⚡│
└─────────────────────────────────────────────────────────────────┘
```

#### Verification Page

```
┌─────────────────────────────────────────────────────────────────┐
│  ⛓️ IMMORTALIZE                                      Verify     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✓ VERIFIED IMMORTALIZATION                                     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │  Note: note1abc123...                                   │    │
│  │  Author: @alice (npub1xyz...)                           │    │
│  │                                                         │    │
│  │  "This is my prediction: Bitcoin hits $500k by 2030"    │    │
│  │                                                         │    │
│  │  ──────────────────────────────────────────────────     │    │
│  │                                                         │    │
│  │  Anchor:     Litecoin                                   │    │
│  │  Block:      2,700,100                                  │    │
│  │  Timestamp:  January 15, 2025 14:32:00 UTC              │    │
│  │  TX:         abc123def456...                            │    │
│  │  Hash:       sha256:789xyz...                           │    │
│  │                                                         │    │
│  │  [View on Block Explorer]  [View on Nostr]              │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### API Specification

#### Endpoints

```
POST /api/v1/immortalize
  Request:
    {
      "note_id": "nevent1..." | "note1...",
      "chain": "ltc" | "btc",
      "payment_hash": "<lightning payment hash>"
    }
  Response:
    {
      "status": "pending" | "confirmed",
      "txid": "...",
      "block": 2700100,
      "timestamp": 1734400000,
      "hash": "sha256:...",
      "proof_url": "https://immortalize.xyz/proof/..."
    }

GET /api/v1/verify/{note_id}
  Response:
    {
      "verified": true | false,
      "immortalizations": [
        {
          "chain": "ltc",
          "block": 2700100,
          "txid": "...",
          "timestamp": 1734400000
        }
      ]
    }

GET /api/v1/status/{txid}
  Response:
    {
      "status": "pending" | "confirmed",
      "confirmations": 6,
      "block": 2700100
    }
```

#### NIP-XX: Immortalization Events

```json
{
  "kind": 10043,
  "pubkey": "<immortalizer pubkey>",
  "created_at": 1734400000,
  "tags": [
    ["e", "<immortalized note id>", "<relay>", "immortalized"],
    ["p", "<original author pubkey>"],
    ["chain", "ltc"],
    ["block", "2700100"],
    ["txid", "abc123..."],
    ["hash", "sha256:789xyz..."]
  ],
  "content": "",
  "sig": "..."
}
```

---

## Technical Architecture

### System Overview

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Web App   │────▶│   API       │────▶│  Worker     │
│   (React)   │     │   (Node)    │     │  (Queue)    │
└─────────────┘     └─────────────┘     └──────┬──────┘
                           │                   │
                           ▼                   ▼
                    ┌─────────────┐     ┌─────────────┐
                    │  Database   │     │  Blockchain │
                    │  (Postgres) │     │  (LTC/BTC)  │
                    └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐     ┌─────────────┐
                    │   Nostr     │     │  Lightning  │
                    │   Relays    │     │   (LND)     │
                    └─────────────┘     └─────────────┘
```

### Component Details

#### Frontend (Web App)

```
Tech: React + TypeScript + Vite
Libraries:
  - nostr-tools (Nostr protocol)
  - @getalby/lightning-tools (Lightning payments)
  - @tanstack/react-query (data fetching)
  - tailwindcss (styling)

Hosting: Vercel / Cloudflare Pages
```

#### Backend (API)

```
Tech: Node.js + TypeScript + Express
Libraries:
  - nostr-tools (fetch notes)
  - lndgrpc / lightning (Lightning)
  - litecoin-js / bitcoinjs-lib (blockchain)
  - bull (job queue)
  - prisma (database ORM)

Hosting: Railway / Fly.io / VPS
```

#### Blockchain Integration

```
Litecoin:
  - Electrum server connection
  - Or: LTC Core node (if self-hosting)
  - Commitment: OP_RETURN with hash

Bitcoin:
  - Electrum server / mempool.space API
  - Same commitment method
  - Higher fees, optional upgrade
```

#### Database Schema

```sql
-- Immortalizations
CREATE TABLE immortalizations (
  id UUID PRIMARY KEY,
  note_id VARCHAR(255) NOT NULL,
  note_hash VARCHAR(64) NOT NULL,
  author_pubkey VARCHAR(64),
  chain VARCHAR(10) NOT NULL,
  txid VARCHAR(64),
  block_height INTEGER,
  block_timestamp TIMESTAMP,
  status VARCHAR(20) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  confirmed_at TIMESTAMP,

  INDEX idx_note_id (note_id),
  INDEX idx_txid (txid),
  INDEX idx_status (status)
);

-- Payments
CREATE TABLE payments (
  id UUID PRIMARY KEY,
  immortalization_id UUID REFERENCES immortalizations(id),
  payment_hash VARCHAR(64) NOT NULL,
  amount_sats INTEGER NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  paid_at TIMESTAMP
);

-- Users (optional, for accounts)
CREATE TABLE users (
  id UUID PRIMARY KEY,
  pubkey VARCHAR(64) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),

  INDEX idx_pubkey (pubkey)
);
```

### Security Considerations

```
1. Input validation
   - Validate note IDs (bech32, hex)
   - Sanitize all inputs
   - Rate limiting

2. Payment verification
   - Verify Lightning payment before processing
   - Idempotency keys prevent double-processing

3. Blockchain operations
   - Use established libraries
   - Verify transactions after broadcast
   - Handle reorgs gracefully

4. API security
   - HTTPS only
   - CORS configuration
   - API key for high-volume access
```

---

## Development Phases

### Phase 1: Foundation (Weeks 1-3)

#### Week 1: Core Infrastructure

```
Tasks:
├── [ ] Set up monorepo structure
├── [ ] Initialize React frontend
├── [ ] Initialize Node.js backend
├── [ ] Set up PostgreSQL database
├── [ ] Configure deployment pipelines
├── [ ] Set up LTC Electrum connection
└── [ ] Basic API skeleton

Deliverables:
├── Working dev environment
├── Database schema deployed
├── Can connect to LTC network
└── CI/CD pipeline working
```

#### Week 2: Nostr Integration

```
Tasks:
├── [ ] Fetch notes from relays
├── [ ] Parse and validate note content
├── [ ] Compute canonical hash
├── [ ] Fetch profiles (kind:0)
├── [ ] Handle multiple relay fallback
└── [ ] Cache relay responses

Deliverables:
├── Can fetch any note by ID
├── Can fetch any profile
├── Consistent hashing implemented
└── Relay redundancy working
```

#### Week 3: Blockchain Anchoring

```
Tasks:
├── [ ] Create LTC transactions with OP_RETURN
├── [ ] Broadcast transactions
├── [ ] Monitor for confirmations
├── [ ] Store proof data
├── [ ] Build verification logic
└── [ ] Handle edge cases (reorgs, failures)

Deliverables:
├── Can anchor hash to LTC
├── Can verify existing anchors
├── Confirmation tracking working
└── End-to-end flow complete (no payments)
```

### Phase 2: Payments & UI (Weeks 4-6)

#### Week 4: Lightning Integration

```
Tasks:
├── [ ] Set up LND node (or use Voltage/Alby)
├── [ ] Generate invoices
├── [ ] Verify payments
├── [ ] Handle payment webhooks
├── [ ] Implement payment queue
└── [ ] Add payment status tracking

Deliverables:
├── Can generate Lightning invoices
├── Can detect payment completion
├── Payment → immortalization flow working
└── Refund handling for failures
```

#### Week 5: Frontend Development

```
Tasks:
├── [ ] Landing page
├── [ ] Note input form
├── [ ] Lightning payment modal (WebLN + QR)
├── [ ] Status/progress display
├── [ ] Verification page
├── [ ] Recent immortalizations feed
└── [ ] Mobile responsive design

Deliverables:
├── Complete user-facing web app
├── Lightning payment working
├── Verification page working
└── Mobile-friendly
```

#### Week 6: Polish & Testing

```
Tasks:
├── [ ] Error handling throughout
├── [ ] Loading states
├── [ ] Edge case testing
├── [ ] Performance optimization
├── [ ] Security audit (basic)
├── [ ] Copy and messaging refinement
└── [ ] Analytics integration

Deliverables:
├── Production-ready MVP
├── All error cases handled
├── Sub-3s response times
└── Ready for beta users
```

### Phase 3: Launch & Iterate (Weeks 7-9)

#### Week 7: Soft Launch

```
Tasks:
├── [ ] Deploy to production
├── [ ] Invite 50-100 beta users
├── [ ] Monitor for issues
├── [ ] Gather feedback
├── [ ] Quick bug fixes
└── [ ] Start documentation

Deliverables:
├── Live production system
├── First real immortalizations
├── Feedback collected
└── Critical bugs fixed
```

#### Week 8: Public Launch

```
Tasks:
├── [ ] Public announcement (Nostr, Twitter)
├── [ ] Submit to Nostr client developers
├── [ ] Create demo content
├── [ ] Press/influencer outreach
├── [ ] Monitor scaling
└── [ ] Respond to feedback

Deliverables:
├── Public launch complete
├── Initial user acquisition
├── Media coverage (Nostr community)
└── Stable under load
```

#### Week 9: Client Integrations

```
Tasks:
├── [ ] Draft NIP for immortalization events
├── [ ] SDK/library for client integration
├── [ ] Reach out to Damus, Amethyst, Primal
├── [ ] Documentation for developers
├── [ ] Example integration code
└── [ ] Support integration partners

Deliverables:
├── NIP draft published
├── JavaScript SDK released
├── At least 1 client integration started
└── Developer docs complete
```

### Phase 4: Growth & Expansion (Weeks 10-12)

#### Week 10: Feature Expansion

```
Tasks:
├── [ ] Thread immortalization (multiple notes)
├── [ ] Bulk immortalization
├── [ ] Bitcoin option (higher fee tier)
├── [ ] User accounts (optional)
├── [ ] History/dashboard for repeat users
└── [ ] API keys for developers

Deliverables:
├── Thread support live
├── Bulk pricing available
├── BTC option available
└── Developer API access
```

#### Week 11: Growth Features

```
Tasks:
├── [ ] Badges/widgets for profiles
├── [ ] Embeddable verification widget
├── [ ] Referral program
├── [ ] Social sharing features
├── [ ] "Immortalized" Nostr badge (NIP)
└── [ ] Leaderboard (most immortalized)

Deliverables:
├── Viral features live
├── Embeddable widgets
├── Referral tracking
└── Social proof mechanisms
```

#### Week 12: Optimization & Scale

```
Tasks:
├── [ ] Performance optimization
├── [ ] Cost optimization (batching)
├── [ ] Monitoring and alerting
├── [ ] Backup and recovery procedures
├── [ ] Documentation complete
└── [ ] Plan next phase (Evidence trails)

Deliverables:
├── System optimized
├── Operational runbooks
├── Ready for 10x scale
└── Roadmap for expansion
```

---

## Go-to-Market

### Target Users

#### Primary: Nostr Power Users

```
Who: Active Nostr users, 1000+ followers
Why: Understand value of permanence
How: Direct outreach, influencer partnerships
Volume: 10,000-50,000 potential users
```

#### Secondary: Nostr Developers

```
Who: Client developers, tool builders
Why: Integrate into their products
How: Developer relations, SDK, documentation
Volume: 100-500 developers → reach their users
```

#### Tertiary: Crypto/Bitcoin Community

```
Who: Bitcoin maxis, crypto natives
Why: Love on-chain verification
How: Twitter, podcasts, conferences
Volume: Broader awareness, future users
```

### Launch Strategy

#### Pre-Launch (Week 6-7)

```
├── Build waitlist
├── Tease on Nostr
├── Recruit beta users (invite-only)
├── Create demo immortalizations
└── Prepare launch content
```

#### Launch Week (Week 8)

```
Day 1: Announce on Nostr (personal account)
Day 2: Twitter announcement
Day 3: Submit to Nostr client devs
Day 4: Hacker News / Reddit
Day 5: Nostr influencer outreach
Day 6: Respond to feedback
Day 7: Week 1 retrospective post
```

#### Post-Launch (Week 9+)

```
├── Weekly feature updates
├── User testimonials
├── Integration announcements
├── Educational content (why permanence matters)
└── Community building
```

### Marketing Channels

| Channel | Effort | Expected Impact |
|---------|--------|-----------------|
| Nostr native | High | Primary acquisition |
| Twitter/X | Medium | Awareness, credibility |
| Developer outreach | High | Integration multiplier |
| Content marketing | Medium | SEO, education |
| Podcasts | Low | Credibility, awareness |

### Messaging

#### Primary Message
> "Make any Nostr note permanent. Forever."

#### Supporting Messages
> "Your note. Your timestamp. Your proof."
> "They can delete relays. They can't delete the blockchain."
> "Immortalize your predictions, your statements, your work."

#### Call to Action
> "Immortalize your first note free" (promotional)
> "500 sats to last forever"

---

## Business Model

### Pricing Structure

| Product | Price (sats) | Price (USD) | Margin |
|---------|--------------|-------------|--------|
| Single note (LTC) | 500 | ~$0.15 | ~$0.10 |
| Single note (BTC) | 2000 | ~$0.60 | ~$0.30 |
| Profile (LTC) | 1000 | ~$0.30 | ~$0.20 |
| Thread ≤10 (LTC) | 2500 | ~$0.75 | ~$0.50 |
| Bulk 100 (LTC) | 25000 | ~$7.50 | ~$5.00 |
| API (per call) | 400 | ~$0.12 | ~$0.08 |

### Cost Structure

| Cost | Monthly | Notes |
|------|---------|-------|
| Hosting | $50-100 | Railway/Fly.io |
| LTC transactions | Variable | ~$0.01-0.05 per tx |
| Lightning node | $20-50 | Voltage or self-hosted |
| Domain/SSL | $5 | Cloudflare |
| Monitoring | $20 | Better Uptime, etc. |
| **Total fixed** | **~$100-200** | Before transaction costs |

### Revenue Projections

#### Conservative

```
Month 1:    500 immortalizations    × $0.10 margin  = $50
Month 3:    2,000                   × $0.10         = $200
Month 6:    10,000                  × $0.10         = $1,000
Month 12:   50,000                  × $0.10         = $5,000/month

Year 1 total: ~$25,000
```

#### Moderate

```
Month 1:    1,000 immortalizations  × $0.10 margin  = $100
Month 3:    5,000                   × $0.10         = $500
Month 6:    25,000                  × $0.10         = $2,500
Month 12:   100,000                 × $0.10         = $10,000/month

Year 1 total: ~$50,000
```

#### Optimistic

```
Month 1:    2,000 immortalizations  × $0.10 margin  = $200
Month 3:    15,000                  × $0.10         = $1,500
Month 6:    75,000                  × $0.10         = $7,500
Month 12:   250,000                 × $0.10         = $25,000/month

Year 1 total: ~$150,000
```

### Path to Profitability

```
Break-even: ~1,500-2,000 immortalizations/month
Target: 10,000+/month by month 6
Profit margin at scale: 60-70% (tx costs minimal)
```

---

## Success Metrics

### North Star Metric

**Total Immortalizations (cumulative)**

```
Month 1:    1,000
Month 3:    10,000
Month 6:    50,000
Month 12:   200,000
```

### Supporting Metrics

#### Acquisition

| Metric | Target (M1) | Target (M6) | Target (M12) |
|--------|-------------|-------------|--------------|
| Unique users | 500 | 5,000 | 20,000 |
| New users/week | 100 | 500 | 1,000 |
| Client integrations | 0 | 3 | 10 |

#### Activation

| Metric | Target |
|--------|--------|
| Visitor → first immortalization | 20% |
| Time to first immortalization | <3 minutes |
| Payment completion rate | 80% |

#### Retention

| Metric | Target (M6) |
|--------|-------------|
| Users with 2+ immortalizations | 40% |
| Users with 10+ immortalizations | 10% |
| Monthly active users | 30% of total |

#### Revenue

| Metric | Target (M6) | Target (M12) |
|--------|-------------|--------------|
| MRR | $2,500 | $10,000 |
| Average revenue per user | $0.50 | $0.75 |
| API revenue % | 10% | 25% |

#### Technical

| Metric | Target |
|--------|--------|
| Uptime | 99.5% |
| Avg response time | <500ms |
| Confirmation time (LTC) | <10 minutes |
| Error rate | <1% |

---

## Risk Assessment

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| LTC network issues | Low | High | BTC fallback, queue retry |
| Lightning payment failures | Medium | Medium | Multiple retry, refund flow |
| Relay unavailability | Medium | Low | Multiple relay fallback |
| Database failure | Low | High | Regular backups, replicas |
| DDoS attack | Medium | Medium | Cloudflare, rate limiting |

### Business Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Low adoption | Medium | High | Strong launch, integrations |
| Competition | Low | Medium | First mover, network effects |
| Regulatory | Low | Low | Simple timestamping service |
| LTC/BTC price volatility | Medium | Low | Price in sats, adjust frequently |

### Operational Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Key person dependency | High | High | Documentation, open source |
| Burnout | Medium | Medium | Realistic scope, automation |
| Support overload | Medium | Low | FAQ, self-service, community |

---

## Resource Requirements

### Team (Minimum Viable)

```
Solo developer: Can build MVP
Ideal small team:
├── 1 Full-stack developer (lead)
├── 1 Part-time designer
└── 1 Part-time community/marketing

Total: 1-2 FTE equivalent
```

### Technology

```
Required:
├── Domain: immortalize.xyz (or similar)
├── Hosting: Railway/Fly.io/Vercel
├── Database: PostgreSQL (managed)
├── LTC node access: Electrum server
├── Lightning: LND (Voltage or self-hosted)
└── Monitoring: Basic (free tier)

Optional:
├── BTC node access (for BTC option)
├── CDN (Cloudflare free tier)
└── Error tracking (Sentry free tier)
```

### Budget

#### Bootstrap Budget ($15K)

```
Development (3 months):
├── Solo developer time: $10K (opportunity cost / contract)
├── Design: $2K
├── Infrastructure: $500
├── Domain/misc: $500
└── Marketing: $2K

Total: $15K
```

#### Comfortable Budget ($25K)

```
Development (3 months):
├── Lead developer: $12K
├── Part-time developer: $5K
├── Design: $3K
├── Infrastructure: $1K
├── Marketing: $3K
└── Contingency: $1K

Total: $25K
```

---

## Timeline

### Gantt Chart (Simplified)

```
Week    1   2   3   4   5   6   7   8   9   10  11  12
        ─────────────────────────────────────────────────
Phase 1 ████████████
  Infra ████
  Nostr     ████
  Chain         ████

Phase 2             ████████████
  Pay               ████
  UI                    ████
  Test                      ████

Phase 3                         ████████████
  Beta                          ████
  Launch                            ████
  Integ                                 ████

Phase 4                                     ████████████
  Features                                  ████
  Growth                                        ████
  Scale                                             ████
```

### Key Milestones

| Milestone | Target Date | Success Criteria |
|-----------|-------------|------------------|
| **M1: Core working** | Week 3 | Can anchor note to LTC |
| **M2: Payments working** | Week 4 | End-to-end with Lightning |
| **M3: MVP complete** | Week 6 | Ready for beta users |
| **M4: Beta launch** | Week 7 | 100 beta users |
| **M5: Public launch** | Week 8 | Public availability |
| **M6: First integration** | Week 10 | Client integration live |
| **M7: 10K immortalizations** | Week 12 | Growth milestone |

---

## Future Roadmap

### Post-Launch Features (Months 4-6)

```
├── Evidence trails (legal/complaint use case)
├── Profile trail (full Blocktrails identity)
├── Subscription plans (unlimited immortalizations)
├── White-label API for clients
└── Mobile app (simple wrapper)
```

### Expansion (Months 7-12)

```
├── Full Blocktrails integration
├── Agent identity features
├── Enterprise offering
├── Additional chain support
└── Hardware wallet integration
```

### Vision (Year 2+)

```
├── Become Nostr standard (NIP accepted)
├── Default in major clients
├── Expand beyond Nostr (any content)
├── Evidence/legal vertical
└── Foundation for Credible Exit ecosystem
```

---

## Appendix A: Technical Specifications

### Hash Computation

```typescript
function computeNoteHash(note: NostrEvent): string {
  // Canonical serialization per NIP-01
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
```

### LTC Commitment Transaction

```typescript
async function createCommitment(hash: string): Promise<string> {
  const tx = new Transaction();

  // Input: funded UTXO
  tx.addInput(utxo.txid, utxo.vout);

  // Output 1: OP_RETURN with commitment
  const commitment = Buffer.from('IMM' + hash, 'utf8'); // 'IMM' prefix + 32 byte hash
  tx.addOutput(Script.buildDataOut(commitment), 0);

  // Output 2: Change back to our address
  tx.addOutput(changeAddress, utxo.value - FEE);

  // Sign and broadcast
  tx.sign(privateKey);
  return broadcast(tx.toHex());
}
```

### Verification Logic

```typescript
async function verify(noteId: string, txid: string): Promise<boolean> {
  // 1. Fetch note
  const note = await fetchNote(noteId);
  if (!note) return false;

  // 2. Compute expected hash
  const expectedHash = computeNoteHash(note);

  // 3. Fetch transaction
  const tx = await fetchTransaction(txid);
  if (!tx) return false;

  // 4. Find OP_RETURN output
  const opReturn = tx.outputs.find(o => o.script.isDataOut());
  if (!opReturn) return false;

  // 5. Extract hash from OP_RETURN
  const data = opReturn.script.getData().toString('utf8');
  if (!data.startsWith('IMM')) return false;
  const storedHash = data.slice(3);

  // 6. Compare
  return storedHash === expectedHash;
}
```

---

## Appendix B: NIP Draft

```markdown
NIP-XX: Immortalization Events

`draft` `optional`

This NIP defines a standard for recording blockchain-anchored timestamps
of Nostr events (immortalizations).

## Event Kind

Kind `10043` is used for immortalization records.

## Tags

- `e` - The event ID being immortalized (required)
- `p` - The pubkey of the original event author (required)
- `chain` - The blockchain used: "btc" or "ltc" (required)
- `block` - The block height containing the anchor (required)
- `txid` - The transaction ID (required)
- `hash` - The hash that was anchored (required)

## Example

{
  "kind": 10043,
  "pubkey": "<immortalizer>",
  "created_at": 1734400000,
  "tags": [
    ["e", "<note id>", "<relay>", "immortalized"],
    ["p", "<author pubkey>"],
    ["chain", "ltc"],
    ["block", "2700100"],
    ["txid", "abc123..."],
    ["hash", "sha256:789xyz..."]
  ],
  "content": "",
  "sig": "..."
}

## Verification

Clients SHOULD verify immortalizations by:
1. Fetching the original event
2. Computing its canonical hash
3. Verifying the hash exists in the claimed transaction
4. Verifying the transaction is confirmed in the claimed block

## Display

Clients MAY display an immortalization indicator (e.g., ⛓️) on events
that have valid immortalization records.
```

---

## Appendix C: Launch Checklist

### Pre-Launch

```
[ ] Production environment deployed
[ ] SSL certificates configured
[ ] Database backups automated
[ ] Monitoring alerts configured
[ ] Error tracking enabled
[ ] Lightning node funded
[ ] LTC wallet funded
[ ] Rate limiting configured
[ ] Legal/ToS page added
[ ] Privacy policy added
[ ] Support email configured
[ ] Social accounts created
[ ] Launch content prepared
[ ] Beta users notified
```

### Launch Day

```
[ ] Final smoke test
[ ] Announce on Nostr
[ ] Announce on Twitter
[ ] Monitor error rates
[ ] Monitor payment success
[ ] Respond to initial feedback
[ ] Fix any critical issues
[ ] Celebrate 🎉
```

### Post-Launch

```
[ ] Daily metrics review
[ ] Weekly retrospective
[ ] User feedback collection
[ ] Bug triage and fixes
[ ] Performance optimization
[ ] Feature prioritization
```

---

*Immortalize: Make any Nostr note permanent. Forever.*

*Project Plan v1.0 — December 2025*
