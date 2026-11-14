# 09. Verify before you trust: reading covenants from the chain

A website showing "verified" is a cache of a check you can run yourself. This file covers how to check a covenant, an address or a token on your own machine, and what the chain can and cannot tell you.

## 1. What the chain shows, and when

- A P2SH address is `aa20 BLAKE2b-256(redeem script) 87`. Before a covenant coin is spent, the chain shows only that hash. The script, its keys and its rules are invisible.
- The script appears in the signature script of the first transaction that spends a coin at that address. From then on anyone can read it.
- So you cannot scan the chain for "covenants that name my key". Covenants reach a user in two ways: they deployed it themselves, or someone gave them the script (a share link, a `.ksm` file, a published source).
- Before depositing into a covenant someone else described to you, get the script (or source plus constructor values plus compiler version), compute the address yourself, and compare.

## 2. kaspa-ksm: a portable covenant file and a local verifier

`kaspa-ksm` (npm, MIT, neutral, no dependency on any website) defines the `.ksm` "Kaspa Spend Map": a covenant's script, its address, its spend paths, its parties and enough to build a spend. With the file and your key you can withdraw using any compatible tool.

```
npx kaspa-ksm verify file.ksm                         # script, hash, address, paths and parties agree
npx kaspa-ksm reproduce file.ksm --silverc ./silverc  # recompile the source: readable rules really are the script
```

Verify a covenant ID (a KIP-20 lineage) on your own machine:

```
npx kaspa-ksm@latest verify <covenant_id> \
  --genesis "https://api.kaspa.org/transactions/<genesis txid>?inputs=true&outputs=true" \
  --source contract.sil --ctor ctor.json \
  --node ws://YOUR_NODE:17110 --sdk ./kaspa-wasm32-sdk/nodejs/kaspa --silverc ./silverc
```

The three steps it runs:

1. **ID.** Recompute the covenant ID from the genesis data (BLAKE2b-256 keyed "CovenantID" over the authorizing outpoint and the authorized outputs), in plain JavaScript, exactly as rusty-kaspa does. Only the real genesis hashes to the ID, so where the data came from does not matter.
2. **Chain.** Your node must show a live coin carrying the ID. Consensus lets an ID exist only if a genesis with exactly that content was valid.
3. **Source.** Your silverc compiles the source with the given constructor values; the result must be the script a genesis output commits to. Compare script bytes, not compiler binary hashes (Rust builds are not bit-identical across machines).

Verdicts:

- **VERIFIED SOURCE**: the ID is real, a live coin exists, and at least one program compiles from the published source.
- **LINEAGE ONLY**: the ID is real; no published source matched.
- **INCOMPLETE**: the ID math checks out, your node did not confirm a live coin.
- **NOT VERIFIED**: the genesis data does not hash to the ID.

An ID proves lineage, not behaviour. Verified source says what the code is, not that it is safe. Names and metadata are whatever the deployer declared.

`ksm program <script hex | file>` compares any script with pinned programs: "identical to X" (byte for byte outside the state window), "differs from X" (a modified copy of a known program: bytes changed, and where), or "not a pinned program" (read its source to know what it does).

## 3. Rules for any verifier you build

- The site is never trusted, only convenient. Every verdict must be reproducible locally against your own node.
- Keep every historical compiler build pinned by hash; covenants deployed with an older compiler need that compiler to reproduce.
- A false "verified" badge is worse than none. Ship the weaker labels first ("lineage only") and add stronger ones only when they are proven.
- Show what was not checked.

## 4. Reading transactions

- api.kaspa.org serves transactions as JSON: `https://api.kaspa.org/transactions/<txid>?resolve_previous_outpoints=light`. Copies fetched without resolving previous outpoints, or older cached copies, can lack input amounts, scripts or covenant IDs. Refetch, or read the spent output from its own transaction.
- A pruned node cannot serve old transactions by id. Keep the transactions you depend on.
- To rebuild a real transaction for testing, its recomputed id must equal the real one. If it does not, a field is missing (sequence, lockTime and input DAA scores are common gaps in REST copies).
- Following the chain live: `getVirtualChainFromBlockV2` with full verbosity gives accepted transactions with each input's UTXO entry (amount, script, covenant ID) and the accepting header's time and DAA score. Wait for enough confirmations before treating an acceptance as final.

## 5. Chain first, database second

- A covenant's balance is what the node says is unspent at its address now. A database's record of deposits is provenance, never balance. Show "checking…" until the node answers.
- A covenant address is reusable: someone can deposit again after it was emptied. "Withdrawn" is a state of the balance (zero), not a permanent status.
- Indexers lag and can be wrong. Before money moves, recheck against a node (your own, or a public one through the `Resolver`): the outpoint is unspent, at the expected address, with the expected covenant ID.
- **A wallet's balance is the sum over all its addresses.** HD wallets put change on new addresses; an explorer's balance for one address is not the wallet's. Track every derived receive and change address, plus the user's own covenant addresses.
- **Follow balances by subscription where you can.** `subscribeUtxosChanged` over wRPC (your own node with `--utxoindex`, or a public node through the `Resolver`) tells you about every coin added or removed at the addresses you track, at once. For a handful of addresses, polling the REST API every few seconds is fine; for many, it is slow, rate-limited and misses things. Tokens are the exception (`05-kcc20-tokens.md` section 12).
- **History sync:** dedupe by transaction id, never by "newest time already stored"; page through the full history rather than the first 100; classify a transaction whose outputs all go to the user's own addresses as a transfer.
- **api.kaspa.org times are already milliseconds** (`block_time`, `accepting_block_time`). Do not multiply by 1000.
- **KAS price:** CoinGecko started refusing some server IPs (403 from its CDN) in Oct 2026. api.kaspa.org serves a KAS price (`/info/price`) as a second source. When every source fails, show "price unavailable"; never a hardcoded number.

## 6. Verifying a payment

- Never trust a client-reported txid. Look the transaction up on your node, confirm the output pays the expected address with the expected amount, and that it is accepted.
- For a merchant flow, say "approved" only after the money is confirmed where it should be (in the escrow or at the merchant's address). Small delays are acceptable; false approvals are not.
