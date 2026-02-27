# xStocks0 Cross-Chain Message Flow

## Payload Structure

```
┌──────────────────────────────────────────────────────────────┐
│  OFT Payload  (82 bytes, big-endian)                         │
├──────────┬──────────┬──────────┬──────────────┬─────────────┤
│ token_id │amount_sd │ recipient│  src_eid     │   nonce     │
│  bytes32 │  uint64  │  bytes32 │   uint16     │   uint64    │
│  32 bytes│  8 bytes │  32 bytes│  2 bytes     │  8 bytes    │
└──────────┴──────────┴──────────┴──────────────┴─────────────┘

token_id   : keccak256(symbol) or registry address as bytes32
amount_sd  : amount in shared decimals (8 dec) — chain-agnostic unit
recipient  : destination address (EVM: left-padded, Solana: pubkey, TON: addr hash)
src_eid    : LayerZero source endpoint ID (Ethereum=30101, Solana=30168, TON=30277)
nonce      : monotonically increasing per-source counter
```

---

## Chain Endpoint IDs (LayerZero V2)

| Chain    | Endpoint ID |
|----------|-------------|
| Ethereum | 30101       |
| Solana   | 30168       |
| TON      | 30277       |

---

## Decimal Conventions

| Chain    | Token Decimals | Conversion to Shared (8 dec) |
|----------|---------------|------------------------------|
| Ethereum | 18            | ÷ 10^10                      |
| Solana   | 9             | ÷ 10^1                       |
| TON      | 9             | ÷ 10^1                       |

Dust (remainder after truncation) is returned to the sender on the source chain.

---

## Supply Invariant

```
canonical_supply_on_eth
  == locked_in_OFTAdapter
  >= Σ minted_on_all_remote_chains

locked_in_OFTAdapter
  == xStock0_minted_on_Solana
   + xStock0_minted_on_TON
   + xStock0_minted_on_[other_chains]
```

---

## Flow 1: Ethereum → Solana

```
User (ETH)                   OFTAdapter (ETH)           LZ Endpoint         xstock0 Program (SOL)
    │                               │                        │                       │
    │── approve(adapter, amt) ──────►│                        │                       │
    │                               │                        │                       │
    │── send(dstEid=30168, ...) ───►│                        │                       │
    │                               │                        │                       │
    │                               │ transferFrom(user, amt)│                       │
    │                               │ _totalLocked += amt    │                       │
    │                               │                        │                       │
    │                               │── endpoint.send() ────►│                       │
    │                               │   {payload, options}   │                       │
    │                               │                        │── deliver message ────►│
    │◄─ emit TokensLocked ──────────│                        │                       │
    │                               │                        │                       │
    │                               │                        │   verify_peer()        │
    │                               │                        │   check_nonce()        │
    │                               │                        │   decode_payload()     │
    │                               │                        │   mint_to(recipient)   │
    │                               │                        │   emit MintEvent       │

Events emitted:
  ETH: TokensLocked(sender, dstEid=30168, recipient, amount, nonce)
  SOL: MintEvent(token_id, src_eid=30101, nonce, recipient, amount_sd, amount_spl)
```

### Fee Handling (ETH → SOL)
1. User calls `quoteSend(30168, amount, options)` to get `nativeFee`.
2. `send()` is called with `msg.value >= nativeFee`.
3. LZ executor on Solana is paid from the native fee pool.
4. Excess ETH is refunded to `refundAddress`.

---

## Flow 2: Solana → Ethereum

```
User (SOL)                 xstock0 Program (SOL)        LZ Endpoint         OFTAdapter (ETH)
    │                               │                        │                       │
    │── send(dst=30101, ...) ──────►│                        │                       │
    │                               │                        │                       │
    │                               │ burn(user_ata, amt)    │                       │
    │                               │ total_minted -= amt    │                       │
    │                               │                        │                       │
    │                               │── emit BurnEvent ─────►│  (relayer picks up)   │
    │◄─ BurnEvent emitted ──────────│                        │                       │
    │                               │                        │── deliver message ────►│
    │                               │                        │                       │
    │                               │                        │   verify_peer()        │
    │                               │                        │   check_nonce()        │
    │                               │                        │   _totalLocked -= amt  │
    │                               │                        │   token.transfer(to)   │
    │                               │                        │   emit TokensUnlocked  │

Events emitted:
  SOL: BurnEvent(token_id, dst_eid=30101, sender, recipient_evm, amount_sd, payload)
  ETH: TokensUnlocked(srcGuid, srcEid=30168, recipient, amount, nonce)
```

### Fee Handling (SOL → ETH)
1. User pays SOL for the LZ executor fee (quoted from LZ SDK).
2. Executor on Ethereum calls `lzReceive()` — no additional ETH required from user.

---

## Flow 3: Ethereum → TON

```
User (ETH)                   OFTAdapter (ETH)           LZ Endpoint         Jetton Master (TON)
    │                               │                        │                       │
    │── send(dstEid=30277, ...) ───►│                        │                       │
    │                               │ lock tokens            │                       │
    │                               │── endpoint.send() ────►│                       │
    │                               │                        │── lz_receive msg ─────►│
    │                               │                        │                       │
    │                               │                        │  verify lz_oracle()   │
    │                               │                        │  check nonce+1        │
    │                               │                        │  mint_jettons(to)     │
    │                               │                        │  total_supply += amt  │
    │◄─ emit TokensLocked ──────────│                        │                       │

Recipient in payload:
  TON uses 256-bit address hashes.
  EVM caller encodes recipient as: sha256(TON_address_raw_bytes) → uint256 → bytes32
  TON master reconstructs the address from the hash.
```

---

## Flow 4: TON → Ethereum

```
User (TON)          Jetton Wallet (TON)    Jetton Master (TON)   LZ Oracle (TON)   OFTAdapter (ETH)
    │                       │                      │                    │                   │
    │── burn(amt,           │                      │                    │                   │
    │   recipient_evm) ────►│                      │                    │                   │
    │                       │ balance -= amt        │                    │                   │
    │                       │── burn_notification ─►│                    │                   │
    │                       │   (amount,            │                    │                   │
    │                       │    from_addr,         │                    │                   │
    │                       │    recipient_evm)     │                    │                   │
    │                       │                       │ total_supply -= amt│                   │
    │                       │                       │── lz_send_notify ─►│                   │
    │                       │                       │   {payload}        │── relay msg ──────►│
    │                       │                       │                    │                   │
    │                       │                       │                    │  verify peer       │
    │                       │                       │                    │  check nonce       │
    │                       │                       │                    │  unlock tokens     │
    │                       │                       │                    │  emit TokensUnlocked│
```

---

## Failure Recovery

### Scenario A: Message Delivered but Execution Fails

LayerZero V2 supports **message retry** via the endpoint's `retryMessage()`.

```solidity
// Ethereum OFTAdapter – expose retry hook
function retryReceive(
    Origin calldata origin,
    bytes32 guid,
    bytes calldata message
) external onlyRole(ADMIN_ROLE) {
    lzReceive(origin, guid, message, address(0), "");
}
```

On Solana, the LZ executor retries the transaction until it succeeds or the nonce is consumed.

### Scenario B: Message Never Delivered (Executor Failure)

Users can call `endpoint.retryMessage()` on the source chain with the original payload to re-trigger delivery. This requires:
- Original `guid` (emitted in `TokensLocked` / `BurnEvent`)
- Original `options` bytes

### Scenario C: Emergency Pause + Recovery

1. Admin calls `pause()` on both adapter and token.
2. No new messages are processed.
3. Admin investigates discrepancy.
4. If `totalLocked` > actual balance: `emergencyWithdraw()` to treasury.
5. Unpause resumes normal operations.

### Scenario D: Nonce Gap (Out-of-Order Delivery)

- Ethereum adapter: nonces are **not** ordered — each `(srcEid, nonce)` is tracked independently via `_usedNonces` mapping.
- Solana program: nonces are **strictly ordered** (`inbound_nonce + 1`). Out-of-order messages are rejected and must be redelivered in correct sequence.
- TON master: same strict ordering as Solana.

This is a deliberate tradeoff: Ethereum is most flexible (parallel delivery), while SVM/TVM enforce ordering for simpler state management.

---

## Message Retry Logic (Reference)

```
retry_interval = [30s, 2m, 10m, 1h, 6h, 24h]  // exponential backoff
max_retries    = 10
after_max      = alert + manual intervention via retryMessage()
```

---

## Security Properties

| Property              | Mechanism                                          |
|-----------------------|----------------------------------------------------|
| Replay protection     | `(srcEid, nonce)` replay map / NonceRecord PDA     |
| Peer authentication   | `peers[srcEid]` whitelist, checked before decode   |
| Supply invariant      | `_totalLocked` counter on Ethereum; `total_minted` on remote |
| Dust elimination      | Shared-decimal truncation before lock/burn         |
| Emergency egress      | `emergencyWithdraw` gated by `whenPaused + ADMIN`  |
| Transfer compliance   | `_restricted` map in xStock ERC20 `_update` hook  |
| Upgrade path          | Admin role can deploy new adapter, migrate `totalLocked` |
