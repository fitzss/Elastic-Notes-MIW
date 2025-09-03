# MVP: Portable Notes Payment Loop

## One-sentence goal
Prove that tiny prepaid notes can move as simple data between people and services, be spent instantly off-chain, and be redeemed on-chain in one cheap batch later.

## What the MVP includes
- **A funded reserve on Ergo** using the Basis contract, which safely holds ERG or another token.
- **A minimal note format in JSON**, signed by the issuer, that says “this is worth X from reserve Y.”
- **A spend record in JSON**, signed by the holder, that says “debit Z from note N.”
- **A small verifier service** that checks signatures, tracks remaining balances, and returns results instantly.
- **A batch redemption script** that submits a list of accepted spends to the Basis reserve once per demo session or once per day.
- **A transport adapter** for sending and receiving notes and spends. Start with Nostr for peer to peer delivery, keep the interface pluggable so Waku or HTTP can be added later.
- **A simple wallet UI** that can receive, show, spend, and redeem notes. It maps to MIW’s “fast”, “backed”, and “non-backed” modes.

## Two user stories the MVP must demonstrate
- **Person to person cash**  
  Alice receives a small backed note, pays Bob instantly, and both can see what is backing the note and any acceptance rules.
- **Machine to service micro-payment**  
  A caller attaches a ten-cent note to an API call. The service verifies and debits two cents locally, returns the result at once, then redeems all accepted spends in one on-chain claim later.

## End-to-end flow
### Mint
- Developer funds the Basis reserve on testnet.
- The server creates a signed JSON note, for example 0.10 unit, and hands it to a user or a demo client.

### Send
- The note travels as data, for example over Nostr, HTTP, QR, or a file. The MVP starts with Nostr.

### Spend
- The payer creates a signed spend record that says “take 0.002 from note ABC.”
- The service verifies the issuer signature on the note, verifies the holder signature on the spend, checks remaining balance in a local table, and replies “paid” immediately.

### Record
- The service stores the tuple note_id and spend_nonce, and the debit amount. It updates remaining_value in its local table.

### Redeem in batch
- Later the service submits one on-chain transaction with a list of unique keys built from note_id and spend_nonce and the total debit amount.
- The Basis reserve verifies that each key is new, then releases the total to the redeemer. This prevents double spend at settlement.

## The data objects
### Note
`note_id, reserve_id, face_value, remaining_value, unit, issuer_pubkey, holder_pubkey, nonce, issuer_sig`

### Spend
`note_id, debit, holder_pubkey, spend_nonce, holder_sig`

### Optional assignment
`note_id, from_pubkey, to_pubkey, assign_nonce, from_sig`  
Lets a note change hands before it is spent.

### Redeem claim
`reserve_id, array of {note_id, debit, spend_nonce}, and tracker_sig if a tracker helps sign the batch`

## Components and how they fit
- **Basis reserve contract**  
  Holds funds. Keeps an authenticated tree of used spend keys to block duplicates. Releases funds only when presented with a valid batch claim.
- **Transfer and verification service**  
  Parses and verifies notes and spends.  
  Maintains a small database of notes, remaining balances, and accepted spends.  
  Exposes two simple endpoints or handlers, for example `receive_note` and `receive_spend`.  
  Builds and submits the redemption transaction.
- **Transport adapter**  
  Minimal interface, for example `send(obj)` and `subscribe(type, handler)`.  
  First adapter uses Nostr events to publish and receive notes and spends.  
  Later adapters can add Waku and HTTP without changing money logic.
- **Wallet UI**  
  Shows balance of face value and remaining value.  
  Shows backing and the reserve link for “backed”.  
  Lets a user accept, spend, and redeem.  
  Exposes simple presets that match MIW.  
  - *Fast*: uses low friction rules.  
  - *Backed*: shows redeemability.  
  - *Non-backed*: uses circle rules and limits.

## Why this MVP works
- **Tiny, instant payments**  
  Every payment rides as data, so there is no on-chain fee per call. Only the batch touches the chain.
- **Clean separation of roles**  
  Issuance and redemption are on-chain for trust and finality. Spending is off-chain for speed and cost.
- **Transport flexibility**  
  Notes are just text, so they can move over Nostr, Waku, HTTP, email, or QR. If one network is blocked, use another. The core money logic does not change.
- **Fits MIW naturally**  
  MIW defines the user experience and acceptance rules. Basis gives secure reserves and redemption. Portable notes tie them together.

## Security model in plain English
- **Authenticity**  
  Notes are signed by the issuer. Spends are signed by the current holder. Fakes fail verification.
- **No reuse of spends**  
  Each spend includes a fresh random number. The reserve keeps a hashed set of used numbers. Duplicate reuse is rejected during redemption.
- **Local limits**  
  The service tracks remaining value and refuses when a note runs out. The wallet can impose per-spend minimums to avoid dust.

## What to show in the demo
- A web page or simple app that lets a user click “Get 0.10 test note.”
- A second page that calls the demo API with “Pay 0.002 and summarize this text,” then shows “Paid, summary here.”
- A status panel that shows accepted spends piling up, then a “Redeem now” button that submits one on-chain batch and shows the transaction link.

## Success criteria
- Spend latency under a second on normal connections.
- At least ten spends included in a single redemption.
- Duplicate spend attempts rejected during redemption.
- Transport adapter can be swapped in code with a small interface.

## Risks and simple mitigations
- **User onboarding**: Provide a faucet for tiny demo notes, backed by a small reserve that you fund once.  
- **Rounding and dust**: Use integer cents, define a minimum spend, and document rounding.  
- **Key management**: Keep demo keys in a secure test vault. For real use, integrate hardware or a secure enclave.  
- **Relay availability**: Use two or more Nostr relays. If a relay is unavailable, fall back to HTTP for the demo so the loop still works.

## What comes next after the MVP
- Add Waku transport for mobile and privacy friendly routing.
- Add blind signatures for private notes if privacy is a goal.
- Publish a tiny SDK for services to accept notes and emit spends.
- Add circle presets in the wallet so communities can publish acceptance rules.

## Why this matters
Right now micro-priced work is hard to sell. Wallet prompts and per-transaction fees break the flow and the economics. This MVP shows a different path. Money moves like data, services respond instantly, and settlement happens cleanly in one batch. That is a foundation MIW can build on for people and for agents.
