# Storage Architecture

> Resolves issue #1192 — Create comprehensive documentation describing the contract
> storage architecture and how data is organized.

## Table of Contents

1. [Overview](#overview)
2. [Storage Tiers](#storage-tiers)
3. [Storage Key Catalog](#storage-key-catalog)
   - [Instance Storage](#instance-storage)
   - [Persistent Storage](#persistent-storage)
   - [Temporary Storage](#temporary-storage)
4. [Data Relationships](#data-relationships)
5. [Storage Diagrams](#storage-diagrams)
6. [Module Map](#module-map)
7. [Migration Guidelines](#migration-guidelines)
8. [Cost Model](#cost-model)

---

## Overview

The **ClipCash NFT** contract is a Soroban smart contract deployed on the Stellar network.
All data is stored on-chain using Soroban's three-tier key-value storage model.
Every storage key is an XDR-serialised enum variant; there are **no string-keyed entries**.

Storage is organised into three logical groups:

| Group | Key enum | Module |
|-------|----------|--------|
| Contract-global config | `DataKey` (lib.rs) / `StorageKey` (storage/keys.rs) | `storage/` |
| Per-token data | `DataKey::Token(token_id)` | `token_storage.rs` |
| Per-owner indexes | `DataKey::Balance(addr)` etc. | `wallet_token_index.rs` |

---

## Storage Tiers

Soroban provides three storage tiers with different cost and TTL semantics.

### Instance Storage

- **Scope**: One record per deployed contract instance.
- **TTL**: Automatically extended when any transaction touches the contract.
- **Use case**: Global config, counters, and flags whose absence would break the contract.
- **Cost**: Cheapest per-byte on-chain; shared across all callers.

### Persistent Storage

- **Scope**: Arbitrary key-value pairs that outlive the transaction.
- **TTL**: Manually bumped; entries expire if not refreshed.
- **Use case**: Per-token data, per-owner balances, approvals, blacklists.
- **Cost**: Higher than instance; TTL bump transactions add incremental cost.
- **Bump constants**: `PERSISTENT_BUMP_THRESHOLD = 172_800 ledgers`,
  `PERSISTENT_BUMP_AMOUNT = 535_680 ledgers` (~30 days).

### Temporary Storage

- **Scope**: Key-value pairs that are automatically deleted after the current
  transaction completes (or optionally survive a configurable number of ledgers).
- **TTL**: Zero by default; use only for within-transaction coordination.
- **Use case**: Reentrancy lock, within-transaction gas accumulators.
- **Cost**: Cheapest overall; no persistent fees.

---

## Storage Key Catalog

### Instance Storage

All instance keys live in `DataKey` (canonical) and are mirrored by `StorageKey`
in `clips_nft/src/storage/keys.rs`.

| Key | Type | Description |
|-----|------|-------------|
| `DataKey::Admin` / `StorageKey::Admin` | `Address` | Contract administrator (single owner). Set once during `init`. |
| `DataKey::PendingOwner` | `Option<Address>` | Nominated new owner awaiting acceptance (two-step transfer). |
| `DataKey::NextTokenId` / `StorageKey::NextTokenId` | `u32` | Auto-increment counter; next token ID to assign on mint. |
| `DataKey::TotalSupply` / `StorageKey::TotalSupply` | `u32` | Live count of minted-minus-burned tokens. |
| `DataKey::Paused` / `StorageKey::Paused` | `bool` | Global pause flag. `true` blocks all user mutations. |
| `DataKey::MintingPaused` | `bool` | Minting-specific pause (transfers still allowed). |
| `DataKey::Name` / `StorageKey::Name` | `String` | Collection name, e.g. "ClipCash Clips". |
| `DataKey::Symbol` / `StorageKey::Symbol` | `String` | Collection ticker symbol, e.g. "CLIP". |
| `DataKey::Signer` / `StorageKey::Signer` | `BytesN<32>` | Ed25519 public key used to verify mint signatures. |
| `DataKey::BackendAddress` | `Address` | Off-chain backend's on-chain identity. |
| `DataKey::PlatformRecipient` / `StorageKey::PlatformRecipient` | `Address` | Recipient of platform fees. |
| `DataKey::PlatformFeeBps` / `StorageKey::PlatformFee` | `u32` | Platform fee in basis points (100 = 1 %). |
| `DataKey::DefaultRoyaltyBps` / `StorageKey::DefaultRoyaltyBps` | `u32` | Default royalty applied when none is set per-token. |
| `DataKey::DefaultRoyaltyAsset` | `Option<Address>` | SEP-0041 token address for royalty payments; `None` = XLM. |
| `DataKey::MintCooldownSeconds` | `u64` | Minimum seconds between mints per wallet. `0` = no cooldown. |
| `DataKey::MaxSupply` / *(implicit)* | `Option<u32>` | Hard cap on total mintable supply; `None` = unlimited. |
| `DataKey::CircuitBreakerEnabled` | `bool` | When `true`, a mint rate limiter is active. |
| `DataKey::CircuitBreakerThreshold` | `u64` | Max mints allowed per window. |
| `DataKey::CircuitBreakerWindowSeconds` | `u64` | Window duration in seconds for the circuit breaker. |
| `DataKey::CircuitBreakerWindowStart` | `u64` | Timestamp when the current window opened. |
| `DataKey::CircuitBreakerWindowCount` | `u64` | Number of mints in the current window. |
| `DataKey::ContractVersion` / `StorageKey::ContractVersion` | `u32` | Semantic version integer; bumped on upgrades. |
| `StorageKey::Config` | `Config` | Consolidated config struct (admin, fees, limits). |
| `StorageKey::SupportedCurrencies` | `Vec<Address>` | List of whitelisted SEP-0041 payment tokens. |
| `StorageKey::CollectionMetadata` | `CollectionMetadata` | Collection-level metadata blob. |
| `StorageKey::ListingCounter` | `u32` | Auto-increment counter for marketplace listing IDs. |
| `StorageKey::MarketplaceConfig` | *(struct)* | Marketplace fee and configuration settings. |
| `StorageKey::PlatformRevenue` | `i128` | Accumulated platform fee revenue. |
| `StorageKey::MaxMetadataSize` | `u32` | Max byte length of a metadata URI. |
| `StorageKey::MetadataVersion` | `u32` | Global metadata schema version used for migrations. |
| `StorageKey::TokenCounter` | `u32` | Total cumulative mints (independent of burns). |

---

### Persistent Storage

Persistent entries are keyed by token ID, address, or a combination.

#### Token Records

| Key | Type | Description |
|-----|------|-------------|
| `DataKey::Token(token_id)` / `StorageKey::Token(token_id)` | `TokenData` | Core token struct: owner, clip_id, soulbound flag, metadata_uri, image, royalty, locked flag. |
| `StorageKey::Metadata(token_id)` | `String` | Canonical metadata URI (may differ from `TokenData.metadata_uri` post-update). |
| `StorageKey::TokenUri(token_id)` | `String` | Custom URI override set by the owner. |
| `StorageKey::Royalty(token_id)` | `Royalty` | Per-token royalty configuration; overrides default. |
| `StorageKey::RoyaltyRecipient(token_id)` | `Address` | Primary royalty recipient for quick lookup. |
| `DataKey::RoyaltyBalance(token_id)` | `i128` | Accumulated unclaimed royalty balance (in asset's smallest unit). |
| `DataKey::Frozen(token_id)` / `StorageKey::FrozenToken(token_id)` | `bool` | When `true`, transfers are blocked for this token. |
| `DataKey::MetadataRefreshTime(token_id)` | `u64` | Ledger timestamp of the last metadata refresh (30-day cooldown). |
| `DataKey::MetadataUpdateCount(token_id)` | `u32` | Number of metadata updates performed on the token. |
| `StorageKey::MetadataUpdated(token_id)` | `bool` | Flag set after the first metadata update. |
| `StorageKey::ClipInfo(token_id)` | `ClipInfo` | Extended clip data (title, description, etc.). |
| `StorageKey::MetadataTimestamps(token_id)` | `MetadataTimestamps` | Creation timestamp + last update timestamp. |
| `StorageKey::ThumbnailUri(token_id)` | `String` | Static thumbnail URI for the clip. |
| `StorageKey::PreviewVideoUri(token_id)` | `String` | Animated preview video URI for the clip. |
| `StorageKey::ViralityScore(token_id)` | `u32` | AI-generated virality score. |
| `StorageKey::SocialPlatform(token_id)` | `u32` | Originating social platform identifier. |
| `StorageKey::VideoSourceId(token_id)` | `u32` | Original video source identifier. |
| `StorageKey::VideoSourceUrl(token_id)` | `String` | Original video source web URL. |
| `StorageKey::Creator(token_id)` | `Address` | Address of the original creator. |
| `StorageKey::RoyaltyHistory(token_id)` | `Vec<RoyaltyPayment>` | Historical royalty payment log. |
| `StorageKey::RoyaltyPercentage(token_id)` | `u32` | Per-token royalty % in basis points. |
| `StorageKey::TokenClipId(token_id)` | `u32` | Reverse map: token_id → clip_id. |
| `StorageKey::TokenOwner(token_id)` | `Address` | Direct owner address record (supplemental to TokenData). |
| `StorageKey::TokenCollection(token_id)` | `u32` | Collection ID this token belongs to. |

#### Clip-to-Token Index

| Key | Type | Description |
|-----|------|-------------|
| `DataKey::ClipIdMinted(clip_id)` / `StorageKey::ClipIdMinted(clip_id)` | `TokenId` | Maps off-chain clip_id to on-chain token_id. Prevents double-minting. |
| `DataKey::MintedClip(clip_id)` / `StorageKey::ClipMinted(clip_id)` | `bool` | Boolean existence check for a clip_id (faster than checking ClipIdMinted). |

#### Per-Owner Indexes

| Key | Type | Description |
|-----|------|-------------|
| `DataKey::Balance(address)` | `u32` | Number of tokens currently owned by `address`. |
| `DataKey::OwnerTokenIndex(address, u32)` | `TokenId` | Ordered list of token IDs for enumeration. Index 0..balance-1. |
| `StorageKey::WalletTokens(address)` | `Vec<TokenId>` | Alternative wallet token index (used by modular wallet_token_index.rs). |
| `StorageKey::OwnerTokens(address)` | `Vec<TokenId>` | Owner portfolio list (used by owner_portfolio.rs). |
| `StorageKey::CreatorTokens(address)` | `Vec<TokenId>` | Creator portfolio list. |

#### Approvals

| Key | Type | Description |
|-----|------|-------------|
| `DataKey::Approved(token_id)` / `StorageKey::Approval(token_id)` | `Address` | Single-token approval: one address permitted to transfer token_id. |
| `DataKey::ApprovalForAll(BytesN<32>)` | `bool` | Compact approval-for-all: key is SHA-256(owner_xdr ‖ operator_xdr). |
| `StorageKey::OperatorApproval(owner, operator)` | `bool` | Full address-pair approval-for-all map (modular approach). |

#### Blacklists

| Key | Type | Description |
|-----|------|-------------|
| `DataKey::BlacklistedClip(clip_id)` / `StorageKey::ClipInfo(clip_id)` | `bool` | Prevents blacklisted clip IDs from being minted. |
| `DataKey::BlacklistedWallet(address)` / `StorageKey::Blacklisted(address)` | `bool` | Prevents blacklisted wallets from minting or receiving transfers. |

#### Signature & Nonce

| Key | Type | Description |
|-----|------|-------------|
| `DataKey::UsedSignature(BytesN<32>)` / `StorageKey::UsedSignature(BytesN<32>)` | `bool` | Marks consumed signature hashes to prevent replay attacks. |
| `DataKey::LastMintNonce(address)` / `StorageKey::Nonce(address)` | `u64` | Per-wallet strictly-increasing nonce for `mint_with_signature`. |
| `StorageKey::ApprovedMinter(address)` | `bool` | Authorised minter role entries. |
| `StorageKey::Administrator(address)` | `bool` | Supplemental admin role entries (multi-admin pattern). |

#### Collection & Marketplace

| Key | Type | Description |
|-----|------|-------------|
| `StorageKey::CollectionSupply(collection_id)` | `u32` | Token count in a collection. |
| `StorageKey::CollectionRegistered(collection_id)` | `bool` | Registration status of a collection. |
| `StorageKey::CollectionMembers(collection_id)` | `Vec<TokenId>` | Tokens belonging to this collection. |
| `StorageKey::Listings(listing_id)` | `Listing` | Active marketplace listing. |
| `StorageKey::Offers(listing_id)` | `Vec<Offer>` | Offers made against a listing. |
| `StorageKey::SaleHistory(token_id)` | `Vec<Sale>` | Historical sale records for a token. |
| `StorageKey::MetadataRecord(uri)` | `bool` | Uniqueness index: prevents duplicate metadata URIs. |
| `StorageKey::MetadataIndex(uri)` | `TokenId` | Maps metadata URI string → token_id. |

#### Withdraw Requests

| Key | Type | Description |
|-----|------|-------------|
| `DataKey::WithdrawXlmRequest` | `WithdrawRequest` | Pending time-locked XLM withdrawal (48-hour delay). |
| `DataKey::LastWithdrawalTime` | `u64` | Timestamp of the most recent completed withdrawal. |

---

### Temporary Storage

| Key | Type | Description |
|-----|------|-------------|
| `DataKey::ReentrancyLock` | `bool` | Reentrancy guard; set on entry, cleared on exit of vulnerable functions. |
| `DataKey::TotalGasMint` | `u64` | Running sum of simulated gas consumed by mint operations. |
| `DataKey::CountMint` | `u64` | Running count of mint transactions. |
| `DataKey::TotalGasTransfer` | `u64` | Running sum of simulated gas consumed by transfer operations. |
| `DataKey::CountTransfer` | `u64` | Running count of transfer transactions. |

---

## Data Relationships

```
                    ┌─────────────────────────────────────────┐
                    │           Instance Storage               │
                    │  Admin, Config, TotalSupply, Paused …   │
                    └────────────────┬────────────────────────┘
                                     │ references
           ┌─────────────────────────┼────────────────────────┐
           │                         │                         │
           ▼                         ▼                         ▼
  ┌─────────────────┐    ┌───────────────────┐    ┌─────────────────────┐
  │  Token(token_id)│    │ ClipIdMinted(clip)│    │  Balance(address)   │
  │  ─────────────  │    │ ─────────────────  │    │  ───────────────     │
  │  owner: Address │◄──►│  → token_id: u32  │    │  → count: u32        │
  │  clip_id: u32   │    └───────────────────┘    └──────────┬──────────┘
  │  metadata_uri   │                                         │ indexed by
  │  royalty        │                                         ▼
  │  is_soulbound   │                           ┌────────────────────────────┐
  │  is_locked      │                           │ OwnerTokenIndex(addr, pos) │
  └────────┬────────┘                           │  → token_id: u32           │
           │ has                                └────────────────────────────┘
           │
     ┌─────┴────────────────────────────────────┐
     │                                           │
     ▼                                           ▼
┌───────────────────────┐          ┌──────────────────────────┐
│  Approved(token_id)   │          │  RoyaltyBalance(token_id)│
│  → operator: Address  │          │  → amount: i128           │
└───────────────────────┘          └──────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │                    Operator Approvals                            │
  │                                                                  │
  │  OperatorApproval(owner, operator) → bool  (modular path)        │
  │  ApprovalForAll(SHA-256(owner||operator)) → bool  (compact path) │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Storage Diagrams

### Mint flow — storage writes

```
mint(to, clip_id, …)
│
├── INSTANCE: NextTokenId ← token_id + 1
├── INSTANCE: TotalSupply ← supply + 1
│
├── PERSISTENT: Token(token_id) ← TokenData { owner: to, clip_id, … }
├── PERSISTENT: ClipIdMinted(clip_id) ← token_id
├── PERSISTENT: MintedClip(clip_id) ← true
├── PERSISTENT: Balance(to) ← balance + 1
├── PERSISTENT: OwnerTokenIndex(to, balance) ← token_id
├── PERSISTENT: TokenIndex(supply) ← token_id   [global enumeration]
│
└── TEMPORARY:  CountMint ← count + 1
                TotalGasMint ← gas + GAS_BASE_MINT
```

### Transfer flow — storage writes

```
transfer(from, to, token_id, …)
│
├── PERSISTENT: Token(token_id).owner ← to
├── PERSISTENT: Approved(token_id) ← remove
├── PERSISTENT: Balance(from) ← balance - 1
├── PERSISTENT: Balance(to) ← balance + 1
├── PERSISTENT: OwnerTokenIndex(to, new_pos) ← token_id
│
└── TEMPORARY:  CountTransfer ← count + 1
                TotalGasTransfer ← gas + GAS_BASE_TRANSFER
```

### Burn flow — storage writes

```
burn(owner, token_id)
│
├── INSTANCE: TotalSupply ← supply - 1
│
├── PERSISTENT: Token(token_id) ← remove
├── PERSISTENT: ClipIdMinted(clip_id) ← remove
├── PERSISTENT: Approved(token_id) ← remove
├── PERSISTENT: Balance(owner) ← balance - 1
└── PERSISTENT: (OwnerTokenIndex gap filled by swap-and-pop)
```

### Approval flow — storage writes

```
approve(caller, operator, token_id)
│
└── PERSISTENT: Approved(token_id) ← operator   (or remove if None)

set_approval_for_all(owner, operator, approved)
│
├── PERSISTENT: OperatorApproval(owner, operator) ← approved  (modular)
└── PERSISTENT: ApprovalForAll(hash) ← approved               (compact)
```

---

## Module Map

The contract storage layer is spread across several modules. Here is the canonical
map from storage concern to source file:

| Concern | Primary module | Supplemental |
|---------|---------------|-------------|
| Storage key enum (modular) | `src/storage/keys.rs` | `src/types.rs` (legacy `DataKey`) |
| Config getter/setter | `src/storage/config.rs` | `src/config.rs` |
| Storage errors | `src/storage/errors.rs` | `src/types.rs` (Error enum) |
| Per-token CRUD | `src/token_storage.rs` | `src/token_owner_storage.rs` |
| Metadata storage | `src/metadata/storage.rs` | `src/token_metadata_storage.rs` |
| Royalty storage | `src/royalty_storage.rs` | `src/nft_royalty_storage.rs` |
| Wallet index | `src/wallet_token_index.rs` | `src/owner_portfolio.rs` |
| Approval storage | `src/token_approval.rs` | `src/operator_approval.rs` |
| Nonce & signature | `src/nonce_storage.rs` | `src/signature_replay_storage.rs` |
| Listing storage | `src/listing_storage.rs` | `src/marketplace/listing_storage.rs` |
| Access guards | `src/storage_guard.rs` | `src/guards/` |
| Admin storage | `src/administrator_storage.rs` | — |
| Minter role storage | `src/minter_role_storage.rs` | — |
| Creator storage | `src/creator_storage.rs` | `src/creator_auth_storage.rs` |

---

## Migration Guidelines

Follow these guidelines whenever a storage layout change is needed in a future
upgrade.

### Additive changes (safe)

1. **New keys** — Add a new variant to `StorageKey` / `DataKey`. Existing data is
   unaffected. Default values are returned by callers that use `.unwrap_or(…)`.
2. **New fields on structs** — Soroban XDR is positional; adding fields at the end
   of a `#[contracttype]` struct is backward-compatible if old data uses
   `Option<T>` for the new field.

### Breaking changes (require migration transaction)

1. **Renaming or reordering key variants** — Existing on-chain entries are keyed by
   XDR-encoded ordinal; changing variant positions corrupts reads. Instead:
   - Add the new key variant alongside the old one.
   - Deploy a migration transaction that reads every old key and writes the new key.
   - Remove the old variant only after migration is complete and verified.

2. **Changing a struct field type** — XDR decoding will fail on old data. Follow
   the same add-migrate-remove pattern.

3. **Changing the instance storage layout** — Use the `ContractVersion` key:
   ```rust,ignore
   let version: u32 = env.storage().instance()
       .get(&StorageKey::ContractVersion)
       .unwrap_or(1);
   if version < 2 {
       migrate_v1_to_v2(&env);
       env.storage().instance().set(&StorageKey::ContractVersion, &2u32);
   }
   ```

### Persistent entry TTL

Persistent entries expire if their TTL is not refreshed. When adding a new
persistent key:

1. Always bump the TTL immediately after writing:
   ```rust,ignore
   env.storage().persistent().set(&key, &value);
   env.storage().persistent().extend_ttl(
       &key,
       PERSISTENT_BUMP_THRESHOLD,
       PERSISTENT_BUMP_AMOUNT,
   );
   ```
2. Add a `keep_alive` call in any read-heavy path so entries renewed
   by off-chain activity do not expire silently.

### Storage cost estimation

Before deploying a migration:
- Estimate new persistent entry count × average entry size.
- Multiply by the current Stellar ledger fee schedule for persistent storage.
- Budget an admin call that performs the migration in a single transaction (or
  batched calls if the dataset is too large for one transaction's fee budget).

---

## Cost Model

| Tier | Write cost | Read cost | TTL strategy |
|------|-----------|-----------|-------------|
| Instance | Low | Low | Auto-extended with every contract call |
| Persistent | Medium | Low | Must be manually bumped; expires without renewal |
| Temporary | Lowest | Lowest | Destroyed at end of transaction (or short TTL) |

**Best practices:**
- Keep hot counters (`TotalSupply`, `NextTokenId`) in **instance** storage.
- Keep per-token data in **persistent** storage; bump TTL on every write.
- Use **temporary** storage only for within-transaction guards (reentrancy lock)
  and accumulators that do not need to survive.
- Avoid unbounded `Vec` values in instance storage; prefer indexed persistent
  entries (`OwnerTokenIndex`, `TokenIndex`) for collections that grow linearly
  with the number of tokens.
