# MNS_SECURITY.md

## 1) Overview

Megabytes Name System (**MNS**) provides:

- **On-chain identity registration** (name → identity public key)
- **Off-chain encrypted messaging** (**MMSG**) for end-to-end chat
- **Off-chain signed presence** (**MMSG_PRESENCE**) for “online” discovery and message routing
- **Network-level anti-abuse protections** to prevent presence/message floods and memory exhaustion

Security goals:

- **Authenticity**: presence announcements and message signatures are attributable to an identity key.
- **Confidentiality**: message content is encrypted end-to-end (E2E).
- **Integrity**: ciphertext and metadata are authenticated (MAC).
- **Spam resistance**: rate-limits, throttles, scoring, and caps reduce abuse impact.
- **DoS resistance**: bounded memory, size checks, and periodic cleanup.

Non-goals / out-of-scope (current):
- Full anonymity guarantees against a global adversary
- Strong Sybil resistance beyond standard discourage/ban logic
- Traffic-analysis resistance

---

## 1.1 Defensive Design & Hardening Measures

MNS and MMSG are designed with a **defensive-first mindset**.  
All off-chain components assume that network inputs may be malformed, malicious, replayed, or intentionally crafted to exhaust resources.

The system enforces **strict bounds, caps, and validation at every layer**.

### Input validation & bounds checking
- All network payloads are validated before parsing.
- Length-prefixed fields (CompactSize vectors, signatures, ciphertexts) are **strictly bounded**.
- Signature vectors are capped to prevent oversized allocations.
- Invalid or malformed frames are rejected early with minimal processing cost.
- Authentication (MAC) is verified **before** any decryption attempt.

### Memory safety & bounded structures
- No unbounded vectors, maps, or caches are allowed.
- All in-memory indexes enforce **hard maximum sizes**.
- Presence index entries and candidate lists are capped.
- Abuse tracking structures use sliding windows with pruning.

### Disk-first inbox design
- The off-chain inbox snapshot database is the **single source of truth**.
- RAM caches are strictly non-authoritative and may be dropped at any time.
- All inbox operations (ACK, delete, prune) operate on disk-backed ordered indexes.
- Atomic deletion removes all associated index entries to prevent orphaned data.

### Deterministic pruning & limits
- Global inbox size is capped.
- Per-sender inbox limits are enforced.
- Time-based TTL pruning removes stale messages deterministically.
- Oldest entries are always pruned first using ordered indexes.

### Rate limiting & abuse resistance
- Per-peer rate limits prevent message flooding.
- Per-identity throttles limit repeated updates from the same identity.
- Burst detection escalates penalties automatically.
- Identity churn detection prevents bypassing limits via key rotation.
- Global message caps protect node-wide resources.

### Fail-safe behavior
- On validation failure, messages are dropped without partial state updates.
- No background threads are required for cleanup.
- Restarting the node always restores a consistent inbox state from disk.

These measures ensure that MNS remains:
- resistant to memory exhaustion attacks
- robust against malformed or malicious inputs
- predictable under high load
- safe to operate continuously without manual intervention

## 2) Threat model

### 2.1 Network attacker
- Flood `MMSG_PRESENCE` to pollute presence index or spam logs
- Rotate many identities (“identity churn”) to bypass throttles / fill caches
- Send malformed frames (vector-wrapped oddities, bad subtype, invalid lengths)
- Force repeated signature failures to waste CPU
- Reconnect repeatedly, or use many IPs

### 2.2 Content attacker
- Forge presence for another identity (blocked by signature)
- Replay old presence packets (time sanity / stale drop)
- Tamper with encrypted messages (MAC validation fails)
- Send messages from unknown senders (allowed; recipient-side policy decides storage/UX)

### 2.3 Resource attacker (DoS)
- Trigger expensive operations repeatedly
- Attempt to grow in-memory maps without bounds

---

## 3) MNS identity registration (on-chain)

### 3.1 Registration format (v1)
An MNS identity is registered using an `OP_RETURN` output that embeds:

- magic `"MNS"`
- version `0x01`
- `name` (normalized lowercase, 3–32 chars)
- `identity_pubkey` (33 bytes typical; some parsing supports 33/65 sanity)

Validation highlights:
- Name is normalized to lowercase and validated with strict allowed characters.
- Duplicate names are rejected.
- A burn requirement enforces a minimum cost:
  - `burn >= MNS_REGISTRATION_BURN` (e.g. 1 MGB)

### 3.2 Identity key id (CKeyID)
Identity identifier used in network payloads:

- `identity_key_id = Hash160(identity_pubkey)`  
  (Bitcoin-style key id, 20 bytes)

### 3.3 Destination resolution (payments)
`DecodeDestinationOrMNS()` resolves either:
- a normal address (DecodeDestination), or
- an MNS name → record → public key → keyid → destination

Implementation notes:
- Uses `WitnessV0KeyHash(keyid)` (P2WPKH bech32) rather than legacy P2PKH.

---

## 4) MMSG_PRESENCE (signed presence announcements)

`MMSG_PRESENCE` announces that an identity is currently “online” and reachable.
This message is **signed** by the identity key and is protected by strict validation and anti-abuse logic.

### 4.1 Presence payload layout
Parsed fields:

- `msg_subtype` (expected `0x1e`)
- `version`
- `identity_key_id` (20 bytes)
- `nTime` (int64, seconds)
- `vchSig` (DER signature, bounded)

Transport note:
Some peers send vector-wrapped frames:
- vRecv = [CompactSize(len)][payload bytes...]
The implementation supports unwrapping such frames before parsing.

### 4.2 Presence signature (authenticity)
Presence signature is verified by rebuilding a deterministic sighash:

- `sighash = Hash( "MNSP" || version || identity_key_id || nTime )`

Then:
- Resolve `identity_pubkey` from the MNS index (registered identity pubkey)
- Verify: `identity_pubkey.Verify(sighash, vchSig)`

Validation / sanity:
- Signature must be present (`sig_len > 0`)
- Signature vector must be bounded (e.g. `sig_len <= 80`)
- Reject invalid identity pubkey or if identity_key_id is not found in the MNS index

### 4.3 Time sanity / replay resistance
Presence is time bounded:
- If `nTime > now + 120 seconds` → penalize / drop
- If `nTime < now - 600 seconds` → drop silently (stale)
- On update, index may keep `nPresenceTime = max(old, nTime)` to avoid time rollback issues

### 4.4 Presence index updates (bounded memory)
Presence index example fields:
- `identity_id`
- `nLastSeen`
- `nPresenceTime`
- `peer_id`, `last_good_peer`
- `candidates` (set of NodeId) with cap (e.g. MAX_CANDIDATES = 8)

Global cap:
- If a new identity would be inserted while `g_mns_presence_index.size() >= PRESENCE_INDEX_MAX_ENTRIES`
  → drop new identity and penalize (prevents memory exhaustion).

---

## 5) Network anti-abuse for MMSG_PRESENCE

We use a two-layer approach:
- **A = Hard** protections (immediate)
- **B = Soft** protections (scoring) that can “convert B→A” when abuse persists

Key properties:
- Sliding scoring window (e.g. 5 minutes)
- Bounded memory per peer
- Periodic cleanup of inactive peers’ state

### 5.1 A) Hard per-peer rate limit
`PresenceRateLimitPeer(nid, now_ms)` enforces a strict rate.
If exceeded:
- immediate `Misbehaving(..., 10, "mmsgpresence rate limit per peer")`
- message dropped / peer discouraged (depending on Core rules)

### 5.2 B) Identity throttle (soft)
`PresenceThrottleIdentity(identity_u, now_ms)` throttles repeated presence updates for the *same identity*.

If throttled:
- Drop the presence update
- Add a **throttle-hit** score event (low weight) with a cap to prevent “+1 spam”:
  - `PRES_THROTTLE_HIT_SCORE_CAP` limits how many throttle hits can contribute within the window

### 5.3 Burst throttle detection (10 hits / minute)
If a peer generates an abnormal number of throttle hits (e.g. ≥ 10 in 60 seconds),
we treat it as an aggressive spam signal.

State:
- `throttle_hit_times_ms` deque in `PresenceAbuseState`
- prune anything older than 60 seconds

Behavior:
- upon reaching threshold, add a large scoring event (e.g. +100)
- handler can apply **instant punishment** (Misbehaving 100) immediately
- optional punish cooldown avoids spamming logs on special peers

### 5.4 Identity churn detection (many unique identities per peer)
Attackers may rotate identities to bypass throttles.

State:
- `identities_last_seen` map< uint160, last_seen_ms >
- pruned in the same sliding window (e.g. 5 minutes)

Behavior:
- if `unique_identities >= PRES_CHURN_UNIQUE_THRESHOLD`
  → apply a churn penalty (e.g. +50) with cooldown:
  - `PRES_CHURN_COOLDOWN_MS`

### 5.5 Sliding-window score + punishment threshold
Scoring model:
- Each peer has a deque of score events (timestamp, points, is_throttle_hit)
- Events are pruned outside the window (e.g. 5 minutes)
- Total score = sum(points) across events

Punishment decision:
- if `score >= PRES_ABUSE_SCORE_THRESHOLD` then:
  - `Misbehaving(*peer, 100, "DISCOURAGE THRESHOLD EXCEEDED: ...")`

Note:
- Core has special rules for **local peers** (127.0.0.1) and **manually connected peers**
  where discourage/ban may not apply; they may be disconnected without being banned.

### 5.6 Cleanup / bounds
To avoid memory growth:
- Cap events per peer: `PRES_ABUSE_MAX_EVENTS_PER_PEER`
- Periodic cleanup: `PresenceAbuseCleanupOldPeers(now_ms)` removes inactive/empty peer states
- Global caps on presence index and candidates list

---

## 6) MMSG (off-chain encrypted messages)

### 6.1 Message identifier (dedup)
To identify and deduplicate messages:
- `msgid = Hash(payload)`

This enables:
- cache-based duplicate suppression in the P2P layer
- inbox/outbox dedup in wallet/hub logic

### 6.2 Cryptographic design summary
MMSG uses:
- Ephemeral ECDH key agreement (secp256k1)
- Key derivation for encryption + authentication keys
- Authenticated encryption via HMAC over header + ciphertext (MAC-then-decrypt)

Important:
- The implementation must verify authentication (HMAC) **before** decrypting.

### 6.3 MMSG v2 payload format (as implemented)
Layout:

--- begin MMSG v2 layout ---
magic[4] = "MMSG"
ver[1] = 0x02
sender_kid[20]        (CKeyID / Hash160(sender identity pubkey))
eph_pubkey[33]        (sender ephemeral pubkey)
nonce[12]             (random nonce)
recipient_kid[20]     (CKeyID / Hash160(recipient pubkey))
ct_len[CompactSize]   (ciphertext length)
ciphertext[ct_len]
tag[32]               (HMAC-SHA256)
sig_sender[CompactSize+bytes]  (identity signature over header+ct_len+ciphertext+tag)
--- end MMSG v2 layout ---

Notes:
- `recipient_kid` is a key-hint to accelerate wallet decryption (fast path).
- `sender_kid` is included for sender identity context and future validation policies.
- `ct_len` uses CompactSize to support robust parsing and prevent truncation ambiguity.
- The tag is computed over: `header_without_magic || ct_len_ser || ciphertext`
- The sender signature currently exists in payload (parsed as a vector), enabling future “signed messages” verification rules.

### 6.4 ECDH and key derivation
ECDH:
- `shared = ECDH(eph_priv, recipient_pubkey)` → 32 bytes

Key derivation:
- `key_enc = SHA256(shared || "MMSG-ENC")`
- `key_auth = SHA256(shared || "MMSG-AUTH")`

### 6.5 Authentication tag (HMAC)
Compute:
- `tag = HMAC_SHA256(key_auth, header || ct_len_ser || ciphertext)`

Decrypt flow must:
1) parse payload (verify structure)
2) derive shared + keys
3) recompute tag
4) constant-time compare tag
5) only then decrypt ciphertext

### 6.6 Encryption stream (current implementation)
The current encryption method behaves like a stream cipher:
- keystream blocks derived by SHA256(key_enc || nonce || counter)
- ciphertext = plaintext XOR keystream

Security note:
- Nonce uniqueness is critical for stream ciphers.
- Ephemeral ECDH per message makes key reuse unlikely; nonce is still randomized.

### 6.7 Wallet decryption strategy (fast path)
Wallet decryption uses `recipient_kid` hint:
- Extract `recipient_kid`
- Try `GetKey(recipient_kid)` fast path
- Descriptor fallback:
  - iterate scripts, find affected keys
  - obtain private provider for pubkey
  - attempt decrypt

Result notes:
- “Recipient key not found in wallet” → message is not for us
- “Key found but authentication failed” → tampered payload or wrong key material

### 6.8 Sender signature (message-level authenticity)
MMSG v2 includes a sender identity signature appended after tag.
Current decryption focuses on recipient authentication (MAC) + confidentiality.
Future policy (recommended):
- Verify sender signature after successful decrypt (optional)
- Resolve sender pubkey from MNS index via `sender_kid`
- Mark sender as “known/verified” in UI when signature passes
- 
## 6.9) Off-chain inbox snapshot database (security & limits)

The MNS off-chain inbox uses a **disk-backed snapshot database** as its
single source of truth.  
In-memory structures are strictly non-authoritative and exist only for UI
comfort during a live session.

This design prevents spam amplification, memory exhaustion, and state
desynchronization across restarts.

---

### 6.9.1 Source of truth

- **Authoritative state**: on-disk snapshot database
- **Non-authoritative state**: RAM cache (`g_mns_inbox`)

The RAM cache may be cleared at any time without data loss.
All persistence, pruning, and acknowledgements operate on disk only.

---

### 6.9.2 Snapshot database key spaces

The snapshot database is organized into three ordered key spaces.

#### a) Main record (`m`)
Stores the full inbox entry.

Key format:
- `m + msgid`

Stored data includes:
- encrypted payload
- `time_received`
- `recipient_kid`
- `sender_kid` (optional)
- `sender_identity` (optional)
- `from_peer`

---

#### b) Time index (`t`)
Ordered index used for TTL and global inbox limits.

Key format:
- `t + time_be + msgid`

Enforced limits:
- **TTL**: 48 hours
- **Global inbox cap**: 2000 messages

Oldest entries are pruned first.

---

#### c) Sender index (`s`)
Ordered index used to cap messages per sender.

Key format:
- `s + sender_kid + time_be + msgid`

Enforced limits:
- **Maximum 50 messages per sender**

When the cap is exceeded, the oldest messages from that sender are removed.

---

### 6.9.3 Atomic deletion and ACK semantics

When a message is acknowledged or deleted, **all related keys are removed
atomically**:

- main record (`m`)
- time index (`t`)
- sender index (`s`), if present

This guarantees:
- no orphaned index entries
- no index inflation
- correct per-sender counts at all times

The RPC:

- `mns_inbox_snapshot_delete`

operates only on the daemon snapshot database and never touches the wallet
database.

---

### 6.9.4 Pruning strategy

Pruning is enforced through two complementary mechanisms.

#### Time-based and global pruning
- Applied via the time index (`t`)
- Enforces TTL (48h) and global cap (2000)
- Deterministic, ordered, and cheap

#### Per-sender pruning
- Applied via the sender index (`s`)
- Enforces max 50 messages per sender
- Executed opportunistically during insertion
- Avoids global sender scans

No background thread is required.

---

### 6.9.5 Anti-spam and abuse resistance

The inbox storage layer participates directly in abuse mitigation:

- Hard bounds on disk growth
- Deterministic pruning order
- No unbounded maps or vectors
- Deduplication by `msgid = Hash(payload)`

Combined with network-level rate limiting, this prevents inbox flooding,
sender amplification attacks, and long-term storage abuse.

---

### 6.9.6 RAM vs disk responsibilities

Disk snapshot database:
- authoritative inbox state
- persistence across restarts
- used for GUI reloads and ACK logic

RAM cache (`g_mns_inbox`):
- live UI updates
- session-local convenience only
- safe to discard at any time

RAM state must never be trusted for correctness.

---

### 6.9.7 Security guarantees

This design guarantees:

- bounded storage growth
- no orphaned indexes
- no sender-based spam amplification
- deterministic cleanup
- safe restart and crash recovery
- predictable GUI behavior

The inbox system is intentionally disk-first and defensive by design.

## 6.10) Messaging rate-limit profiles

MNS messaging enforces multiple concurrent rate limits to ensure
spam resistance, fairness, and predictable resource usage.

The limits below describe the recommended **“Normal chat, ultra strict”**
profile used by default.

---

### 6.10.1 Snapshot storage limits (disk)

These limits apply to the off-chain inbox snapshot database.

- Maximum messages per sender: **50**
- Maximum total snapshot size: **2000 messages**
- Snapshot TTL: **48 hours**

Older messages are pruned deterministically using ordered indexes.

---

### 6.10.2 Sender-based message rate limits

Applied when the sender identity (`sender_kid`) is known.

- Short burst limit: **5 messages per 10 seconds**
- Sustained limit: **30 messages per 10 minutes**

Purpose:
- Prevent sender-based spam
- Limit automated flooding from a single identity
- Still allow normal human chat patterns

---

### 6.10.3 Peer-based message rate limits

Applied per connected peer (NodeId), regardless of identity.

- Short burst limit: **50 messages per 10 seconds**
- Sustained limit: **200 messages per 10 minutes**

Purpose:
- Prevent a single peer from flooding messages
- Limit abuse from misbehaving or compromised nodes

---

### 6.10.4 Global message rate limits

Applied across all peers and identities.

- Short burst limit: **200 messages per 10 seconds**
- Sustained limit: **1000 messages per 10 minutes**

Purpose:
- Protect global resources
- Prevent coordinated flood attacks
- Ensure predictable CPU and memory usage

---

### 6.10.5 Enforcement model

- All limits are enforced independently
- Hitting any limit causes message drop
- Severe or repeated violations may trigger peer penalties
- Rate-limit checks are cheap and bounded

This layered approach ensures that:
- no single sender can dominate the inbox
- no peer can overwhelm the node
- no coordinated attack can exhaust disk or memory

---

## 7) Operational security (hub/wallet)

Recommended operational rules:
- Import/store only messages whose `recipient_kid` belongs to the wallet
- Allow “unknown sender” messages (sender identity may not be in local contacts yet)
- Dedup using `msgid = Hash(payload)` to avoid reprocessing loops
- Avoid re-decrypting already decrypted messages in UI (cache results by msgid)

---

## 8) Tuning parameters (current defaults)

Typical parameters (example values):
- Presence scoring window: `PRES_ABUSE_WINDOW_MS = 5 minutes`
- Punish threshold: `PRES_ABUSE_SCORE_THRESHOLD = 100`
- Throttle-hit cap: `PRES_THROTTLE_HIT_SCORE_CAP = 10` (per 5 minutes)
- Burst threshold: `PRES_THROTTLE_HITS_PER_MIN_BAN = 10` (per 60 seconds)
- Churn unique identities threshold: `PRES_CHURN_UNIQUE_THRESHOLD = 25`
- Churn cooldown: `PRES_CHURN_COOLDOWN_MS = 60 seconds`
- Max events per peer: `PRES_ABUSE_MAX_EVENTS_PER_PEER = 512`
- Presence index max entries: `PRESENCE_INDEX_MAX_ENTRIES` (bounded)

---

## 9) Limitations / future hardening

Planned improvements:
- Replace custom SHA256 stream cipher with a standard AEAD (ChaCha20-Poly1305 or AES-GCM)
- Optional mandatory sender signature verification (identity authenticity for chat UI)
- Add anti-flood for MMSG payloads (not only presence):
  - per-peer limits, global caps, and dedup
- Optimize MNS pubkey lookup (avoid scanning full MNS list on each presence)
- More metrics and rate-limited logs for production deployments
- Stronger Sybil / eclipse mitigations at network layer (beyond discourage/ban)
