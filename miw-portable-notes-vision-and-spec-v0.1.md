# Vision 

## Title: MIW + portable notes: payment as data for people and agents

Kushti, reading your MIW sketch alongside my Elastic Notes draft, I think we are pointing at the same missing capability. People and agents need a way to move tiny amounts of value quickly, sometimes privately, and without custodians or heavy UX. Today they cannot do this at the scales they need.

MIW frames the wallet side: fast, backed, and non backed. The note idea frames the money object as a small signed packet that moves like data. Together the vision is simple cash that works for communities and machines, that anyone can spin up on demand.

The core idea is payment as data. A note is a signed JSON that says it is worth X from reserve Y, with optional metadata and rules. It can ride in an API call, over Nostr, over Waku, by QR, or as a file. Money logic stays the same while the delivery path can change. That split makes the system durable and practical. Builders do not need to change apps to change transports. It also solves a clear market problem. Developers and agents who price work in micro cents can charge per action without wallets, popups, or per call gas. Communities can circulate credit inside circles with clear acceptance rules and redeem to reserves only when they need to settle.

MIW maps cleanly onto one note model. Fast is the same note used as prepaid credit for speed. Backed is the same note with visible reserve links for redeemability. Non backed is the same note accepted inside a circle by trust and limits. Circles management is where acceptance presets live, so users and merchants can publish what they accept and why. The wallet only needs a few verbs. Receive a note. Spend a note. Redeem a note. View backing and liabilities.

To keep MIW and the note model coherent, the backbone is small and clear. Define a portable note schema that survives any transport and any wallet view. Lock down the off chain loop so it is always mint, attach, verify, debit, batch redeem. Keep reserves on chain with simple rules. Keep spending off chain so it is instant and cheap. Treat transport as a plug. Start with Nostr if the first goal is peer to peer delivery without central servers. Add Waku, HTTP, and offline paths later without touching money semantics.

Two stories prove the job to be done. Person to person cash inside a circle using a small backed or trusted note. Machine to service payment where a caller attaches a ten cent note to an API call, the service verifies and debits locally, and later redeems in one on chain batch. Same note, same rules, same wallet verbs.

If this framing is useful, I can align my write up to MIW and propose a first cut of the note schema and acceptance presets that match fast, backed, and non backed. Or, if transport first is better, I can sketch mint, transfer, and redeem events on Nostr so the wallet, the server, and relays speak the same messages.

## Implementation spec 

## Title: MIW, Basis, and a minimal path to off chain transfer

### Intent: describe a simple way MIW can support off chain spending with on chain settlement, using Basis for reserves and redemption, and a pluggable transport starting with Nostr.

**Objects to pass around**

**Credit object**
```
{
  "note_id": "uuid-or-hash",
  "reserve_id": "ergo-box-or-token-id",
  "face_value": "0.10",
  "remaining_value": "0.10",
  "unit": "USDcents",
  "issuer_pubkey": "hex",
  "holder_pubkey": "hex",
  "nonce": "32B",
  "issuer_sig": "sig-over(note_id||reserve_id||face_value||unit||nonce)"
}
```

**Spend record**
```
{
  "note_id": "same",
  "debit": "0.002",
  "holder_pubkey": "hex",
  "spend_nonce": "32B",
  "holder_sig": "sig-over(note_id||debit||spend_nonce)"
}
```

**Optional assignment**
```
{
  "note_id": "same",
  "from_pubkey": "hex",
  "to_pubkey": "hex",
  "assign_nonce": "32B",
  "from_sig": "sig-over(note_id||to_pubkey||assign_nonce)"
}
```

**Batch redeem claim**

```
{
  "reserve_id": "id",
  "items": [
    {"note_id": "A", "debit": "0.002", "spend_nonce": "n1"},
    {"note_id": "B", "debit": "0.004", "spend_nonce": "n2"}
  ],
  "tracker_sig": "sig-over-concat(items)"
}
```

**Double spend defense with Basis**

At redemption, insert key blake2b256(note_id||spend_nonce) into the Basis AVL tree. Duplicates fail. Basis releases the total after proof checks. On chain stays simple. Off chain stays fast.

## MVP loop

* Mint a small note from a funded Basis reserve, for example 0.10

* Send it with a normal request or message

* Verify and debit off chain, return the result immediately

* Batch redeem accepted spends to the reserve at the end of the day

## Transport plan

Start with Nostr for peer to peer delivery and relay diversity. Keep a tiny interface so transport can be swapped later.

- Event kinds

  - 30100 publish note

  - 30101 assign holder

  - 30102 publish spend

  - 30103 publish redeem claim or pointer to the on chain tx

- Code interface

  - send(obj) and subscribe(type, handler)

  - First adapter uses Nostr relays

  - Later add Waku or HTTP without touching money logic

## MIW UI mapping

* Current status: show face value, remaining value, redeemable value by reserve, and liabilities from unredeemed spends

* My Wallets: backed uses Basis reserves, non backed uses trust rules off chain, fast is a low friction preset

* Circles management: whitelist issuer keys, allow person to person transfers via assignment

## Open points to align

* Units and rounding, use integer cents

* Minimum spend size to avoid dust

* Fees at redemption, flat or proportional, who pays

* One tracker key or a set of tracker keys for redundancy

* Privacy roadmap, blind signatures later

## Proposed next steps

* Finalize the JSON fields and Nostr kinds

* Extend the current server to verify and debit notes and to speak Nostr events

* Fund a Basis reserve on testnet

* Demo the full loop from mint to spend to batch redeem
