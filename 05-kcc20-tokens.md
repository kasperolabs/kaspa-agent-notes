# 05. KCC-20 tokens: detect, pin, read metadata, trace lineage

How to recognize a KCC-20 token on chain, read its name and supply, follow its family, and decide what it can do to a holder. Learned by indexing every KCC-20 token on mainnet from launch day (Oct 2026) through Nov 2026, across several launchpads.

Specs: KCC-1, KCC-20, KCC-23 and related documents at github.com/kaspanet/kccs. Reference programs: github.com/argent-lang/kcc20-reference (KCC20, TokenSeed, PublicMint; the pins below are from commit c8a0871).

## 1. What a token is on chain

- A KCC-20 token is a **family of coins ("cells") sharing one KIP-20 covenant ID**. The covenant ID is fixed at the token's genesis and every cell of the token carries it.
- Each cell is a P2SH coin whose redeem script is a token program with the cell's state inside it: `program = prefix ‖ STATE ‖ suffix` (the KCC-1 template layout). The state holds the amount, the owner and the owner's scheme, borrow settings and an extension commitment.
- Each cell also carries some KAS (0.5 to 1 KAS is common). That KAS belongs to the cell, not to the holder's spendable balance.
- **Identity is the covenant ID. Nothing else.** Name, ticker and image are whatever the deployer declared. Anyone can launch a token with the same name, ticker and image. The program a cell runs gives it a label (what the program can do), not an identity: any covenant can start a genesis that holds cells of a pinned program.

## 2. Recognizing a program (pinning)

The KCC-1 TemplateHash identifies a program independent of the state inside it:

```
TemplateHash = BLAKE3( le64(len(prefix)) ‖ prefix ‖ le64(len(suffix)) ‖ suffix )
```

`le64` is an 8-byte little-endian length, BLAKE3 unkeyed with a 32-byte output. (Some older notes say BLAKE2b. It is BLAKE3.)

Rules for a pin:

- A pin is an exact prefix and suffix, byte for byte. A program one byte off is not the build.
- Pin from a real mainnet spend: the revealed redeem script must hash to the spent output's address (`aa20 BLAKE2b-256(script) 87`), and the state you read must splice back into the program and hash to that same address.
- Keep each pin with where it came from (txid, input index) and what it is.

Builds pinned from mainnet spends (Nov 2026):

| Program | TemplateHash prefix | Notes |
|---|---|---|
| KCC-20 reference cell | `9703112e` | 4,031 bytes, 145-byte state starting at byte 1 |
| Reference program, other input/output limits | `ff3ae732`, `3c1a50ad` | same layout, different limits |
| KRON older cell, 2,433 bytes | `2ed46a7e` | older 46-byte layout, section 7 |
| KRON older cell, 2,732 bytes | `8097c96f` | same layout, also refuses an all-zero owner on type 2 |

The reference cell's transfer and delegator entries show up as dispatch tags `79c71c23` and `fd3ef14a`. A tag found inside pushed data proves nothing (a minting program that carries the cell suffix as an argument contains those bytes too). Decode by the **last push** of the signature script (the program), then by its tag.

Pinning everything you know and ignoring the rest misses every new launcher: each brings its own pools, minters and sometimes its own cell variant. Index everything that moves a known token; let pins decide trust, not visibility (section 10).

## 3. Verifying a cell before you spend it or count it

An indexer answer is a list of candidates. Before spending or showing a balance as final:

1. The P2SH of the cell's redeem script equals the cell's address.
2. The outpoint `txid:vout` is unspent on a node you trust (your own, or a public node through the `Resolver`) at that address and carries the covenant ID you expect.
3. The state you read sits inside the redeem script, and the program around it is a pinned build.

If any check fails, do not spend it and do not count it.

Reader rules learned on mainnet (Oct 2026):

- **Treat an owner that is a covenant as the normal case.** A token's unsold supply is often owned by a pool or curve covenant (scheme 0x04), which has no address. Never derive an address from an owner that is not a key scheme, and never treat "the owner did not decode as an address" as "not a KCC-20 coin". Another public indexer answered "not available" for a token launched this way.
- **One transaction can carry several covenant families.** A genesis can bind a curve coin, a token cell and change. Only outputs whose script matches a token program are cells; other bound P2SH outputs belong to someone else's covenant. Skip them without logging an error.
- A by-owner API that answers only `kaspa:q` addresses cannot show cells owned by a covenant ID or a script hash. If your wallet or app holds tokens in a covenant, the indexer needs a by-covenant-owner query.

## 4. Finding the genesis

From any cell, follow the family input backwards, whatever program it runs, and stop at the transaction whose outputs carry the covenant ID while none of its inputs do. That is the KIP-20 genesis.

- The genesis's **authorizing input** (the one that authorizes the bound outputs) is not always input 0.
- REST copies of transactions can lack input covenant IDs (older copies, or copies fetched without resolving previous outpoints). When a field is missing, read the spent output from its own transaction. Do not walk back on a guess: a pool transaction that also moves another token's cell can send the walk down the wrong history.
- You can recompute the covenant ID from the genesis content (BLAKE2b-256 keyed "CovenantID" over the authorizing outpoint and the authorized outputs) and compare. Only the real genesis hashes to the ID. `kaspa-ksm` does this (`09-verify-and-read-the-chain.md`).

## 5. Metadata (name, ticker, decimals, image)

- **KCC-23**: metadata lives in the transaction spent by the genesis's authorizing input (the "prep" transaction), as a payload of `0x00` followed by JSON. Field names follow KCC-0021: `name`, `ticker` / `symbol`, `decimals`, `image`, `image_hash`, `image_b64`. Keep unknown keys verbatim.
- A prep transaction can have more than one output (an endowment beside the anchor). Accept that.
- Some launchers bind the prep output itself to a launcher covenant born in the same transaction. Accept a bound prep output only when that covenant is born there (its authorizing input spends an output without that covenant ID). Refuse continuations.
- Not every launcher writes KCC-23:
  - KRON launches carry no name on chain. KRON publishes names in its own registry and a signed token list. Take a listed name only with chain evidence (the listed genesis matches yours, decimals match), label it "named by KRON, not on chain", and verify the list's signature against KRON's published key. Never take a placeholder ticker (`STD-…`) as a name.
  - KForge writes `KFORGE6` followed by JSON into the launch transaction that spends the genesis output; accept it only when its `token` field equals the token's own covenant ID.
- On-chain metadata always wins over a listed name.
- Best practice for anyone launching: put metadata and the image (or its hash) on chain at genesis, hash-verifiable, no links an indexer must fetch, no manual registration.
- Description and website fields are an attack surface (phishing links, script tags). Treat every string as hostile; show links as text.

## 6. Supply and creator

- **Fixed-supply tokens: supply = the sum of every genesis cell.** A genesis can bind several outputs, and not every bound output is a cell (seeds, launchers). Prove each cell by hash: rebuild `prefix ‖ state ‖ suffix` with the claimed state, hash it, compare with the output's address. On one launchpad the second genesis cell (a constant pushed by the curve program) was never spent and never shown in a witness; it was only found by rebuilding it from numbers the declaring program pushes. Until every bound output is proven, say "at least N".
- **Minter families start at zero** and add every mint (recipients are readable from the mint arguments and verifiable against the outputs).
- Circulating = supply minus cells sent to provably unspendable owners (burn).
- **Creator** = the address the authorizing input spent. When the authorizing input is itself a covenant (a launchpad's launcher), the creator is the genesis's first plain-key input. Getting this wrong attributed 31 tokens to a launcher instead of their creators.

## 7. KRON's older cell layout (read this before trusting `ownerScheme`)

Two KRON builds use an older 46-byte state at byte 0 of the program (no prefix):

| Offset | Bytes | Field |
|---|---|---|
| 0 | `20` + 32 | owner identifier |
| 33 | `01` + 1 | identifier type |
| 35 | `08` + 8 | amount, unsigned little-endian (top bit of byte 43 must be 0) |
| 44 | `01` + 1 | is_minter |

The identifier type is the cell's own owner rule, **not** the reference program's owner scheme:

| Type | Rule | Owner |
|---|---|---|
| 0 | a 65-byte signature by the owner key | `kaspa:q…` of the key |
| 1 | some input's script is `0000 aa20 <owner> 87` | a P2SH |
| 2 | some input's covenant ID equals the owner | a covenant (KRON's curve or launcher) |
| 3 | some input's script is `0000 20 <owner> ac` (the holder's own plain coin is spent in the same transaction) | `kaspa:q…` of the key |

Reading type 3 as the reference scheme 0x03 ("script hash") hid every KRON holder's coins from a wallet. **Branch on the layout, never on the scheme number alone.** Type 3 checks no signature of its own: see the SIGHASH warning in `01-wallet-safety.md`.

Spending these cells needs a builder written for this layout. The reference KCC-20 builder does not fit it.

## 8. Owner and borrow schemes in the reference program

- Owner schemes seen: 0x00 a Schnorr key (`kaspa:q`), 0x01 a P2PKH (BLAKE3 of a key, signed with key plus signature), 0x02 an ECDSA P2PKH, 0x03 a P2SH script hash, 0x04 a covenant ID. Check the KCC-20 spec for the authoritative list.
- **A cell owned by the P2SH hash (0x03) of a stateful covenant strands after the covenant's first state change** (the address changed). Own such cells by covenant ID (0x04).
- **A cell owned by a covenant ID moves whenever that covenant is spent in the same transaction.** The covenant's own rules must protect it (`04-covenant-design-patterns.md` section 5).
- Borrow schemes (amount threshold, signature guard, hash chain) let others deposit into a cell without the owner's signature. Trade-off seen in practice: each deposit re-creates the cell at a new outpoint, so a pending spend of the old cell becomes invalid and must be rebuilt, and a griefer can block spends by depositing one unit repeatedly (they pay a fee each time; nothing is stolen). A covenant that holds a borrow-enabled cell can be bricked by a stranger (attack BOR-01).

## 9. What a token can do to holders (labels)

Label programs by **provable powers, not intent**. Use the same rule for everyone, including your own tokens.

| Label | Meaning |
|---|---|
| standard | runs a pinned reference build; no admin powers |
| issuer-controlled | runs a pinned issuer program; list its powers (can freeze, can burn, can mint, can seize) |
| unverified | runs a program nobody pinned; may contain rules that let others move coins |
| hidden-powers | an accepted transaction showed a power the program does not declare |
| trap | an accepted transaction showed an outsider moving holders' value |

- Never write "safe", "scam", "fraud" or "malicious". Write the fact and its evidence (txid).
- A label only stands if it can be rerun: same program bytes, same analyzer version, same answer.
- "Keyless" spends on a curve or pool usually mean the pool selling its own inventory to whoever pays, which is what a pool does. Check whose coins moved before alarming anyone.
- An issuer that can take coins from holders' wallets deserves its own badge, shown up front. A regulated stablecoin is supposed to say it can freeze.

## 10. Names, lookalikes and the official list

- Normalize names before comparing: Unicode NFKC, Cyrillic and Greek lookalikes folded to Latin, digit swaps (0/O, 1/I), accents dropped, letters and digits only. `NАCHO` with a Cyrillic А is a copy of `NACHO`.
- Keep a reserved list: majors (Bitcoin, Kaspa, Ethereum, wrapped assets), stablecoin names (USD, USDT, USDC and bridged variants), and existing community tokens from earlier standards. A launchpad should refuse them without explaining why.
- A launched name should be blocked for everyone except its original launcher, who may launch numbered siblings.
- "Official" should come from a manual list with evidence per entry: the project's own announcement URL, a screenshot, a note, a date. Show the evidence. A project page naming a covenant ID can itself be copied; record what you checked.
- **Holder counts lie.** Airdrops and dust inflate them. Count voluntary holders (addresses that signed a spend of the token themselves, or bought on a market). Exclude zero-amount owners. Label holders that are covenants (a pool, a sale contract) instead of warning about concentration.
- Lookalike signals (unsolicited sends to many new addresses, targeting the real token's holders, dust amounts, shared funders among creators, near-miss domains in metadata) are facts about addresses and behaviour, never verdicts about people. One signal alone is noise: a legitimate airdrop looks like dusting. Warn only on combined signals. Never warn on the official token itself: its name signals point at copies of it. A first version of one such watch flagged the real NACHO because ordinary wallet transfers looked like distribution.

## 11. Markets and prices

- An app that only shows a token price can take it from a public source (kascov.io's pool-based figures, or CoinGecko for the tokens it lists) and name that source. The rules below are for building your own price index.
- Your own index's prices must come from on-chain trades you can verify by txid. No invented USD price.
- Generic trade detection works on any DEX: in one accepted transaction, token units move one way between two parties and KAS moves the other way. The counterparty is a covenant whose token delta and KAS delta have opposite signs. Charge the network fee to whoever funded it, outside the trade.
- Exclude from the headline price: same-direction flows (liquidity add or remove), ambiguous transactions, wash trades (same key both sides), thin pools, plain wallet-to-wallet swaps.
- Trust in tiers: reviewed programs, observed programs that meet evidence thresholds (several distinct traders, several fills, no single print far from the last), candidates listed only. Always say which tier and program a price came from.
- 24-hour figures must be rolling (now minus 24 hours), not hour-aligned windows. Keep snapshots so statistics can be checked later.

## 12. Sending tokens

- Build transfers with the builder for the cell's exact program. Token outputs must balance with inputs; each output carries its own KAS; programs cap amounts per output and outputs per transaction.
- The holder signs their own plain KAS inputs (for fees) with SIGHASH_ALL. Leave the token inputs' witnesses exactly as built.
- After broadcasting, tell your indexer about the transaction, or its ledger lags until its scanner catches up.
- A partial send creates a change cell, which takes KAS from the sender (about one cell's worth) beyond the fee.
- **Your address subscription will not see token arrivals.** A cell is a P2SH coin whose address is the hash of its own program and state; it never touches the holder's `kaspa:q` address, and transfers carry no payload. `subscribeUtxosChanged` on the holder's address misses them. Learn arrivals from an indexer (for example, the indexer POSTs an HMAC-signed hook to the wallet's backend, which then re-reads holdings from the indexer and verifies them) or by following the virtual chain yourself.
- **Merging cells gives KAS back.** Every cell carries its own KAS, so a holder with many small cells has KAS stuck in them; one merge of six cells returned 1.29 KAS (mainnet, Oct 2026). Offer one "combine" action for all of them, and refuse a merge of a single cell into itself, which only burns a fee.
- **Variants need their own builder.** A launcher's cell variant with other input and output limits (a 10-in/10-out build) could not be merged by the reference builder; it went out one cell per transaction. KRON's older layout needs its own sender entirely (section 7). Pick the builder by the cell's pinned program, never by "it looks like KCC-20".
- **A send to a covenant ID that no coin carries strands the cell.** No script can check that the receiving covenant exists. Prove it by spending: include one of the receiver's own coins (its deposit entry) as an input in the same transaction, so the receiver's script runs.
- Wallets that do not recognize a program show nothing. A token that "disappeared" from a wallet is usually an indexer that does not read its program, not a lost token.

## 13. Public data sources (Nov 2026, may change)

- kascov.io: an independent KCC-20 and covenant indexer with an open JSON API at `https://kascov.io/data/<network>/…` (docs at kascov.io/#/dev). Its decoder is open source (github.com/Knitser/kascov, MIT).
- `https://scan.kasperolabs.com`: a security-oriented view (program labels, issuer powers, lookalike signals, change facts) with a free badge API: `GET https://scan.kasperolabs.com/api/kcc20/safety/<covenantId>/badge`. Docs: scan.kasperolabs.com/docs and docs/scan-badges in github.com/kasperolabs/silverscript-studio.
- api.kaspa.org: raw transactions (`/transactions/<txid>?resolve_previous_outpoints=light`). Check fields before relying on them: some copies omit input covenant IDs.
- rusty-kaspa has a covenant-ID UTXO index in development (PR #1152, `GetUtxosByCovenantIds`, node flag `--covenant-index`). Not in a release at the time of writing, and an empty ID list is an error, so it cannot discover covenants you don't already know.
