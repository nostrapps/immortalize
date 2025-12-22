# Recipe: Building Nostr + Bitcoin Apps

A guide for LLMs and developers to build client-side apps using Blocktrails, did:nostr, and Preact/HTM.

## Stack Overview

```
┌─────────────────────────────────────────┐
│  Single HTML File (no build process)   │
├─────────────────────────────────────────┤
│  Preact + HTM (UI)                      │
│  @noble/curves (Bitcoin crypto)         │
│  @noble/hashes (SHA256, etc)            │
│  @scure/base (bech32 encoding)          │
├─────────────────────────────────────────┤
│  Nostr Relays (data layer)              │
│  Mempool API (blockchain data)          │
└─────────────────────────────────────────┘
```

## 1. HTML Boilerplate

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>My App</title>
  <style>
    /* CSS here - no external files needed */
    * { box-sizing: border-box; margin: 0; padding: 0; }
    :root {
      --bg: #fafafa;
      --surface: #ffffff;
      --text: #1a1a1a;
      --primary: #6366f1;
    }
    body { font-family: system-ui, sans-serif; background: var(--bg); }
  </style>
</head>
<body>
  <div id="app"></div>
  <script type="module">
    // All JS here - ESM imports from CDN
    import { html, render, useState, useEffect } from 'https://esm.sh/htm/preact/standalone';
    import { secp256k1, schnorr } from 'https://esm.sh/@noble/curves@1.4.0/secp256k1';
    import { sha256 } from 'https://esm.sh/@noble/hashes@1.4.0/sha256';
    import { bytesToHex, hexToBytes } from 'https://esm.sh/@noble/hashes@1.4.0/utils';
    import { bech32, bech32m } from 'https://esm.sh/@scure/base@1.1.6';

    function App() {
      const [count, setCount] = useState(0);
      return html`<button onClick=${() => setCount(c => c + 1)}>Count: ${count}</button>`;
    }

    render(html`<${App} />`, document.getElementById('app'));
  </script>
</body>
</html>
```

## 2. Nostr Integration

### Connect to Relays

```javascript
const DEFAULT_RELAYS = [
  'wss://nos.lol',
  'wss://relay.primal.net',
  'wss://purplepag.es',
  'wss://relay.nostr.band'
];

async function fetchFromRelays(filter, relays = DEFAULT_RELAYS) {
  return new Promise((resolve, reject) => {
    let resolved = false;
    const sockets = [];
    const cleanup = () => sockets.forEach(ws => { try { ws.close(); } catch (e) {} });

    const timeout = setTimeout(() => {
      if (!resolved) { resolved = true; cleanup(); reject(new Error('Timeout')); }
    }, 8000);

    for (const relay of relays) {
      try {
        const ws = new WebSocket(relay);
        sockets.push(ws);
        ws.onopen = () => ws.send(JSON.stringify(['REQ', 'sub', filter]));
        ws.onmessage = (e) => {
          const msg = JSON.parse(e.data);
          if (msg[0] === 'EVENT' && msg[2] && !resolved) {
            resolved = true;
            clearTimeout(timeout);
            cleanup();
            resolve(msg[2]);
          }
        };
      } catch (e) {}
    }
  });
}
```

### Fetch Profile (Kind 0)

```javascript
async function fetchProfile(pubkeyHex) {
  const event = await fetchFromRelays({ kinds: [0], authors: [pubkeyHex], limit: 1 });
  return JSON.parse(event.content);
}
```

### Fetch Note (Kind 1)

```javascript
async function fetchNote(eventId) {
  return await fetchFromRelays({ ids: [eventId], limit: 1 });
}
```

### Sign and Publish Event

```javascript
function computeEventHash(event) {
  const canonical = JSON.stringify([
    0,
    event.pubkey,
    event.created_at,
    event.kind,
    event.tags,
    event.content
  ]);
  return bytesToHex(sha256(new TextEncoder().encode(canonical)));
}

async function createSignedEvent(privateKey, kind, content, tags = []) {
  const pubkey = bytesToHex(secp256k1.getPublicKey(privateKey, true).slice(1));
  const event = {
    pubkey,
    created_at: Math.floor(Date.now() / 1000),
    kind,
    tags,
    content: typeof content === 'string' ? content : JSON.stringify(content)
  };
  event.id = computeEventHash(event);
  event.sig = bytesToHex(await schnorr.sign(event.id, privateKey));
  return event;
}

async function publishEvent(event, relays = DEFAULT_RELAYS) {
  for (const relay of relays) {
    try {
      const ws = new WebSocket(relay);
      await new Promise((resolve, reject) => {
        const timeout = setTimeout(() => { ws.close(); reject(); }, 5000);
        ws.onopen = () => {
          ws.send(JSON.stringify(['EVENT', event]));
          clearTimeout(timeout);
          setTimeout(() => { ws.close(); resolve(); }, 500);
        };
      });
    } catch (e) {}
  }
}
```

## 3. did:nostr Integration

### Encode/Decode

```javascript
// Hex pubkey to npub
function encodeNpub(hexPubkey) {
  const bytes = hexToBytes(hexPubkey);
  const words = bech32.toWords(bytes);
  return bech32.encode('npub', words);
}

// npub to hex
function decodeNpub(npub) {
  if (/^[a-f0-9]{64}$/i.test(npub)) return npub.toLowerCase();
  if (npub.startsWith('npub1')) {
    const { words } = bech32.decode(npub);
    const bytes = bech32.fromWords(words);
    return bytesToHex(new Uint8Array(bytes));
  }
  throw new Error('Invalid pubkey');
}

// Format as DID
function toDidNostr(hexPubkey) {
  return `did:nostr:${encodeNpub(hexPubkey)}`;
}

// Link to resolver
function getNostrProfileUrl(hexPubkey) {
  return `https://nostr.eu/${encodeNpub(hexPubkey)}`;
}
```

### Display Pattern

```javascript
// In your UI, show pubkeys as clickable DIDs:
html`
  <a href=${getNostrProfileUrl(pubkey)} target="_blank">
    ${toDidNostr(pubkey).slice(0, 32)}...
  </a>
`
```

## 4. Blocktrails (Bitcoin Anchoring)

### Core Concept

Blocktrails uses P2TR key tweaking to commit data to Bitcoin:

```
tweak = SHA256(data_hash) mod curve_order
new_privkey = old_privkey + tweak
new_pubkey = old_pubkey + tweak × G
```

The resulting pubkey encodes a commitment to the data.

### Key Tweaking

```javascript
const CURVE_ORDER = secp256k1.CURVE.n;

function tweakPrivkey(privkeyHex, dataHash) {
  const privKey = BigInt('0x' + privkeyHex);
  const tweak = BigInt('0x' + dataHash) % CURVE_ORDER;
  const newPrivKey = (privKey + tweak) % CURVE_ORDER;
  return newPrivKey.toString(16).padStart(64, '0');
}

function tweakPubkey(pubkeyHex, dataHash) {
  // Ensure x-only (32 bytes)
  const xOnly = pubkeyHex.length === 66 ? pubkeyHex.slice(2) : pubkeyHex;

  // Convert to point
  const point = secp256k1.ProjectivePoint.fromHex('02' + xOnly);

  // Compute tweak point
  const tweakBigInt = BigInt('0x' + dataHash) % CURVE_ORDER;
  const tweakPoint = secp256k1.ProjectivePoint.BASE.multiply(tweakBigInt);

  // Add points
  const newPoint = point.add(tweakPoint);
  const newPubkey = newPoint.toHex(true).slice(2); // x-only

  return { tweakedPubkey: newPubkey, tweak: dataHash };
}
```

### P2TR Address Generation

```javascript
function pubkeyToP2TRAddress(xOnlyPubkeyHex, network = 'tbtc4') {
  const pubkeyBytes = hexToBytes(xOnlyPubkeyHex);
  const words = convertBits([...pubkeyBytes], 8, 5, true);
  words.unshift(1); // Witness version 1
  const prefix = network === 'btc' ? 'bc' : network === 'ltc' ? 'ltc' : 'tb';
  return bech32m.encode(prefix, words);
}

function convertBits(data, fromBits, toBits, pad) {
  let acc = 0, bits = 0;
  const result = [], maxv = (1 << toBits) - 1;
  for (const value of data) {
    acc = (acc << fromBits) | value;
    bits += fromBits;
    while (bits >= toBits) { bits -= toBits; result.push((acc >> bits) & maxv); }
  }
  if (pad && bits > 0) result.push((acc << (toBits - bits)) & maxv);
  return result;
}
```

### Taproot Signing (BIP341)

```javascript
function taggedHash(tag, ...data) {
  const tagHash = sha256(new TextEncoder().encode(tag));
  return sha256(new Uint8Array([...tagHash, ...tagHash, ...concatBytes(...data)]));
}

function computeTaprootSighash(tx, inputIndex, prevouts) {
  // Simplified - see full implementation in index.html
  const epoch = new Uint8Array([0]);
  const hashType = new Uint8Array([0]); // SIGHASH_DEFAULT

  // Hash prevouts, amounts, scriptpubkeys, sequences
  const shaPrevouts = sha256(concatBytes(...prevouts.map(p => p.txid_vout)));
  const shaAmounts = sha256(concatBytes(...prevouts.map(p => p.amount)));
  const shaScriptPubkeys = sha256(concatBytes(...prevouts.map(p => p.scriptPubKey)));
  const shaSequences = sha256(concatBytes(...prevouts.map(p => p.sequence)));
  const shaOutputs = sha256(serializeOutputs(tx.outputs));

  return taggedHash('TapSighash',
    epoch, hashType, /* ...other fields... */
    shaPrevouts, shaAmounts, shaScriptPubkeys, shaSequences, shaOutputs
  );
}

async function signTaprootInput(tx, inputIndex, privateKey, prevouts) {
  const sighash = computeTaprootSighash(tx, inputIndex, prevouts);
  const signature = await schnorr.sign(sighash, privateKey);
  return bytesToHex(signature); // 64 bytes, no sighash flag for DEFAULT
}
```

## 5. Mempool API Integration

```javascript
function getApiBase(chain) {
  return {
    'tbtc4': 'https://mempool.guide/testnet4/api',
    'btc': 'https://mempool.guide/api',
    'ltc': 'https://litecoinspace.org/api'
  }[chain] || 'https://mempool.guide/testnet4/api';
}

async function fetchUTXOs(address, chain) {
  const res = await fetch(`${getApiBase(chain)}/address/${address}/utxo`);
  return res.json();
}

async function fetchTx(txid, chain) {
  const res = await fetch(`${getApiBase(chain)}/tx/${txid}`);
  return res.json();
}

async function broadcastTx(txHex, chain) {
  const res = await fetch(`${getApiBase(chain)}/tx`, {
    method: 'POST',
    body: txHex
  });
  if (!res.ok) throw new Error(await res.text());
  return res.text(); // Returns txid
}

function getExplorerUrl(chain, txid) {
  const base = {
    'tbtc4': 'https://mempool.guide/testnet4',
    'btc': 'https://mempool.guide',
    'ltc': 'https://litecoinspace.org'
  }[chain];
  return `${base}/tx/${txid}`;
}
```

## 6. Nostr Login Patterns

### Browser Extension (NIP-07)

```javascript
async function loginWithExtension() {
  if (!window.nostr) throw new Error('No Nostr extension found');
  const pubkey = await window.nostr.getPublicKey();
  return { pubkey, canSign: true };
}

// Sign with extension
async function signWithExtension(event) {
  return await window.nostr.signEvent(event);
}
```

### Direct nsec Input

```javascript
function loginWithNsec(nsecOrHex) {
  const privateKey = nsecOrHex.startsWith('nsec1')
    ? decodeNsec(nsecOrHex)
    : nsecOrHex;
  const pubkey = bytesToHex(secp256k1.getPublicKey(privateKey, true).slice(1));
  return { pubkey, privateKey };
}

function decodeNsec(nsec) {
  const { words } = bech32.decode(nsec);
  const bytes = bech32.fromWords(words);
  return bytesToHex(new Uint8Array(bytes));
}
```

## 7. UI Patterns with Preact/HTM

### State Management

```javascript
function App() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [data, setData] = useState(null);

  const handleAction = async () => {
    setLoading(true);
    setError(null);
    try {
      const result = await someAsyncOperation();
      setData(result);
    } catch (e) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  };

  return html`
    <div>
      ${error && html`<div class="error">${error}</div>`}
      ${loading && html`<div class="loading">Loading...</div>`}
      ${data && html`<div class="result">${JSON.stringify(data)}</div>`}
      <button onClick=${handleAction} disabled=${loading}>
        ${loading ? 'Working...' : 'Do Thing'}
      </button>
    </div>
  `;
}
```

### Conditional Rendering

```javascript
html`
  ${condition && html`<div>Shown if true</div>`}
  ${condition ? html`<div>True</div>` : html`<div>False</div>`}
  ${items.map(item => html`<div key=${item.id}>${item.name}</div>`)}
`
```

### Forms

```javascript
const [input, setInput] = useState('');

html`
  <input
    type="text"
    value=${input}
    onInput=${e => setInput(e.target.value)}
    placeholder="Enter something"
  />
`
```

## 8. Common Utilities

```javascript
// Truncate string with ellipsis
function truncate(str, len = 16) {
  if (!str || str.length <= len) return str;
  return str.slice(0, len / 2) + '...' + str.slice(-len / 2);
}

// Time ago
function timeAgo(timestamp) {
  const seconds = Math.floor(Date.now() / 1000) - timestamp;
  if (seconds < 60) return 'just now';
  if (seconds < 3600) return Math.floor(seconds / 60) + ' min ago';
  if (seconds < 86400) return Math.floor(seconds / 3600) + ' hr ago';
  return Math.floor(seconds / 86400) + ' days ago';
}

// Concat Uint8Arrays
function concatBytes(...arrays) {
  const total = arrays.reduce((sum, arr) => sum + arr.length, 0);
  const result = new Uint8Array(total);
  let offset = 0;
  for (const arr of arrays) { result.set(arr, offset); offset += arr.length; }
  return result;
}

// Write integers (little-endian)
function writeUint32LE(n) {
  const buf = new Uint8Array(4);
  buf[0] = n & 0xff; buf[1] = (n >> 8) & 0xff;
  buf[2] = (n >> 16) & 0xff; buf[3] = (n >> 24) & 0xff;
  return buf;
}

function writeUint64LE(n) {
  const buf = new Uint8Array(8);
  const big = BigInt(n);
  for (let i = 0; i < 8; i++) buf[i] = Number((big >> BigInt(i * 8)) & 0xffn);
  return buf;
}
```

## 9. Project Structure

For a single-page app:

```
my-app/
├── index.html      # Main app (all-in-one)
├── verify.html     # Secondary page (optional)
├── docs.html       # Documentation (optional)
├── README.md       # Project overview
├── CLAUDE.md       # AI development guide
└── RECIPE.md       # This file
```

## 10. Deployment

### GitHub Pages

1. Create repo with `gh-pages` branch
2. Push HTML files to `gh-pages`
3. Site live at `https://username.github.io/repo-name`

```bash
git checkout -b gh-pages
git add *.html README.md
git commit -m "Initial release"
git push -u origin gh-pages
```

### No Build Required

The entire app runs client-side with ES modules from CDN. No npm, no webpack, no build step.

---

## Example Apps to Build

1. **Notary** - Timestamp any text on Bitcoin
2. **Proof of Authorship** - Anchor creative works
3. **Identity Backup** - Snapshot your Nostr profile
4. **Contract Signer** - Multi-party agreement anchoring
5. **Archive** - Preserve tweets/posts to Bitcoin

---

*Built with Blocktrails, did:nostr, and Preact/HTM*
