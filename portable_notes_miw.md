# Portable Notes + MIW: one clear explainer and an MVP we can build

## What it is
A note is like a tiny gift-card code in a text file. It says “this is worth 10¢ from this specific reserve” and it’s signed so anyone can verify it’s real.  

A reserve is a locked box on a blockchain (Basis on Ergo) that holds real funds. The box only opens when it sees a valid batched claim listing the spends that happened.

## Why do it this way
- Pay tiny amounts (e.g., 2¢) instantly and cheaply.
- No wallet popups, no per-call gas.
- A note is just text, so it travels over any channel: Nostr, Waku, HTTP, email, QR, even a USB stick.
- Services total accepted spends and touch the chain once per batch.

## How one payment works
### Mint
Fund a reserve on chain and create a signed 10¢ note (a small JSON object).

### Send
Pass the note to a person or attach it to an API call. It’s just data.

### Spend
To pay 2¢, send a tiny spend record that says “debit 2¢ from note A” and sign it.  

The service verifies signatures and deducts 2¢ in its local DB. Reply is immediate.

### Redeem (batch)
Later, the service submits one on-chain claim totaling all accepted spends.  

The reserve checks each spend has a unique one-time number (nonce) and releases the total.

## What keeps it safe enough
- **Authenticity**: notes and spends are signed; fakes fail verification.
- **No reuse**: every spend has a fresh nonce; the reserve keeps a hashed set of used nonces (AVL tree). Duplicates are rejected at settlement.
- **Local limits**: services track remaining value and refuse once a note is empty.

## How it maps to MIW (Magic Internet Gold Wallet)
- **Fast** = same note used like prepaid credit for speed.
- **Backed** = same note, but the wallet shows the linked reserve so holders can redeem.
- **Non-backed** = same note, accepted inside a circle via trust rules/limits.

Wallet verbs: **Receive note • Spend note • Redeem note • View backing & liabilities.**

## What makes it different from cards or typical crypto
|                   | Cards/Stripe | Lightning   | ERC-20 tx   | Portable note |
|-------------------|--------------|-------------|-------------|---------------|
| **Per-call cost** | High         | Route risk  | Gas         | Zero (off-chain) |
| **UX prompts**    | Frequent     | Often       | Wallet      | None          |
| **Settlement cadence** | Per call | Per call    | Per call    | Batch once    |
| **Transport flexibility** | Low   | Medium      | Medium      | High (text)   |

## Where it helps
- Per-API-call payments for AI/LLM, data, GPU, or any microservice.
- Person-to-person cash in communities.
- Offline/spotty networks via QR/files that sync later.

## The MVP (what to build and why)
### Goal (one sentence)
Show that tiny prepaid notes can move as simple data, be spent instantly off-chain, and be redeemed on-chain in one cheap batch.

### Components
- **Basis reserve (Ergo)** — holds funds; maintains an insert-only AVL tree of used spend keys.
- **Note format (JSON)** — signed by issuer; says “worth X from reserve Y”.
- **Spend record (JSON)** — signed by current holder; says “debit Z from note N”.
- **Verifier service** — checks signatures, tracks balances, replies instantly.
- **Batch redemption script** — submits unique (note_id || spend_nonce) keys + total to Basis.
- **Transport adapter** — send/subscribe interface; start with Nostr; keep pluggable for Waku/HTTP.
- **Simple wallet UI** — receive/show/spend/redeem; presets for fast / backed / non-backed.

### Data objects (concise)
- **Note**: `note_id, reserve_id, face_value, remaining_value, unit, issuer_pubkey, holder_pubkey, nonce, issuer_sig`
- **Spend**: `note_id, debit, holder_pubkey, spend_nonce, holder_sig`
- **(Optional) Assign**: `note_id, from_pubkey, to_pubkey, assign_nonce, from_sig` (move a note before spending)
- **Redeem claim**: `reserve_id, [{note_id, debit, spend_nonce}…], tracker_sig?`

### End-to-end flow
- **Mint** — fund Basis; issue signed 0.10 note.
- **Send** — deliver the note (Nostr first; HTTP/QR also fine).
- **Spend** — holder submits signed debit; service verifies & replies “paid”.
- **Record** — service stores (note_id, spend_nonce, debit) and updates remaining.
- **Redeem** — submit one batch with unique spend keys; Basis pays out; duplicates fail.

### Minimal code slice (illustrative)
```json
{
  "note_id": "A1...",
  "reserve_id": "R1...",
  "face_value": "0.10",
  "remaining_value": "0.10",
  "unit": "USDcents",
  "issuer_pubkey": "...",
  "holder_pubkey": "...",
  "nonce": "...",
  "issuer_sig": "..."
}
```

```bash
# spend 2¢ from note A1
curl -X POST https://server.example/spend -H "Content-Type: application/json" -d '{
  "note_id":"A1","debit":"0.02",
  "holder_pubkey":"...","spend_nonce":"...","holder_sig":"..."
}'
```

## Demo flow to show
1. Mint a 10¢ note.
2. Make five payments of 2¢ each; responses are instant.
3. Run batch redeem; show one on-chain transaction and updated reserve.
4. Show payer’s remaining value 0¢ and the service’s payout received.

## Success criteria
- All spends verified off-chain and under a second.
- One on-chain redemption for all spends.
- Duplicate spends rejected by nonce check.
- Swapping transport (Nostr → HTTP/Waku) does not change money logic.

## Open alignment points (good places to collaborate)
- Units/rounding: use integer cents; define min spend to avoid dust.
- Fees: flat vs proportional; who pays at redemption.
- Tracker keys: single vs set (R6 tree) for redundancy.
- Privacy roadmap: add blind signatures later for private notes.
- Circle presets: acceptance rules for non-backed mode in MIW.

## How this ties back to MIW
- MIW provides the user experience (fast/backed/non-backed) and acceptance rules.
- Basis provides reserves + redemption and double-spend protection at settlement.
- Portable notes provide the off-chain cash that moves like data.  

Together, you get cash-like speed and UX with on-chain finality — suitable for people and agents.

## Next steps (practical)
- Finalize the JSON note/spend fields and Nostr kinds/tags.
- Extend the current server to verify/debit and speak Nostr.
- Fund a testnet reserve and wire up batch redemption.
- Ship the demo page (mint → five spends → one redemption) and publish the spec so others can plug in.

---

That’s the whole picture in one place: the concept, why it matters, how it works, and the exact MVP to prove it.
