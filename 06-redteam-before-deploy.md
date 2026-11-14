# 06. Red-team every .sil file before it holds real money

A covenant cannot be patched after it is funded. The only time to find a hole is before. This file describes the workflow, built around `kaspa-redteam` (github.com/kasperolabs/kaspa-redteam, MIT), and what it does and does not cover.

## 1. The tool

`kaspa-redteam` compiles SilverScript in process, finds an honest spend for every path, then throws a library of attacks at those spends. Every hostile transaction is signed for real and judged by rusty-kaspa's own consensus validator (`kaspa-consensus` 2.1.0, mainnet constants). A finding is reported only when consensus **accepted** a transaction the attack expected to be refused, and it ships with that transaction so anyone can replay it.

- In the browser, nothing to install: **https://redteam.kasperolabs.com** (drop one or more `.sil` files).
- On the command line:

```
cargo install --git https://github.com/kasperolabs/kaspa-redteam kaspa-redteam

kaspa-redteam vault.sil                       # attack it
kaspa-redteam vault.sil -v                    # every attack plus the spend map
kaspa-redteam vault.sil --args deploy.json    # your real constructor values
kaspa-redteam vault.sil --args deploy.json --compiled build.json   # your compiler's exact bytes
kaspa-redteam treasury.sil fund.sil -v        # several files that work together, as one project
kaspa-redteam fetch <txid>... -o real.json    # real transactions as a baseline
kaspa-redteam --baseline real.json            # attack those transactions
kaspa-redteam replay findings/*.json          # re-judge saved findings
kaspa-redteam list                            # the attack library
kaspa-redteam vault.sil --json                # for CI; exit 0 clean, 1 findings, 2 error
```

No node and no separate `silverc` needed for a covenant run.

## 2. The workflow, in order

1. **Compile with your pinned compiler.** Fix every error and warning. Keep source, constructor values and compiler commit together.
2. **Run the red team with your real values** (`--args deploy.json`). Default values can hide or create findings that your deployment does not have.
3. **Read the spend map first (`-v`).** For each path: who can spend it, where the honest spend paid, the earliest lock value consensus accepted. This is the tool's understanding of your contract. If it does not match the sentences you wrote down (`04-covenant-design-patterns.md` section 1), the contract is not what you think.
4. **Treat "unexplored" as untested, never as safe.** A path the search could not spend was not attacked. Give `--arg` values the search could not guess, or a baseline file of honest transactions from your own harness or from chain (`fetch`).
5. **Fix every finding, or write down why it does not matter for this deployment.** Each finding says what consensus accepted, how that happens in practice, and the fix pattern.
6. **Run again until clean and fully explored.** The verdict "PASSED: 0 FINDINGS" means attacks ran, none got through and every path was explored. "0 FINDINGS, N PATHS UNEXPLORED" and "NOTHING TESTED" are not passes.
7. **Attack the bytes you deploy.** If your build pipeline pins a compiler, pass its output with `--compiled`. A contract already on chain keeps the bytecode its compiler made; test it through its real transactions with `--baseline`.
8. **Test contracts that work together as one project** (several `.sil` files in one run). A contract that is clean alone can be exploited through a partner.
9. **Spend every path on testnet** (or with a minimal mainnet amount). The engine does not run mempool policy (section 5).
10. **Deploy small first.** Raise the amount after real spends behave.
11. **For a design that holds other people's money, get one independent review.** Give another model or person a written packet (the sentences, the contracts, the results) and nothing else, cold. Triage every finding in a table (real, by design, text only) and give that table to the next reviewer so it looks elsewhere.

## 3. Findings and their fix patterns

| ID | What got through | Typical fix |
|---|---|---|
| SIG-01 | an invalid signature on a signed path | `require(checkSig(...))`, not a bare `checkSig(...)` |
| SIG-02 | the attacker's own key and signature | the key comes from the constructor or state, not from the caller |
| ST-01 | the attacker's key written into the next state | require the current owner's signature before rewriting the owner |
| SEC-01 | a revealed secret or reading copied from the mempool | add the claimer's signature, or fix the whole spend |
| ORC-04 | a reading signed by the attacker's own key | pin the oracle key in the constructor |
| OUT-01 | same value, attacker's script | pin `tx.outputs[i].scriptPubKey` |
| OUT-02 | output cut to 10% | pin `tx.outputs[i].value` against the coin's value |
| OUT-03 | a later output skimmed | check every output that carries value, or `tx.outputs.length` |
| OUT-04 | leftover after fixed payouts kept by the builder | tie payouts to `tx.inputs[this.activeInputIndex].value` |
| FEE-01 | outputs halved, the rest left as fee | `require(out >= in - maxFee)` |
| IN-01 | two covenant coins satisfied by one set of outputs | `require(tx.inputs.length == 1)` or per-input output indexing |
| IN-02 | attacker coin at input 0 | read `this.activeInputIndex`, never `tx.inputs[0]` |
| FAM-01 to FAM-10 | token family issues: minting too much, to the wrong owner, twice, outside the family; a second genesis; handing out the mint power; taking tokens without paying; sweeping coins sent to the covenant | check amounts, owners, counts and bindings of new coins against what was paid or proven |
| BOR-01 to BOR-07 | borrow paths on held token coins: bricking, moving, taking KAS, rewriting settings | hold borrow-off cells; in a token program, check owner, amount growth, KAS and settings on every borrow |

The examples folder of the repo has every one of these as a broken and a fixed contract.

## 4. Real bugs this workflow caught

- **MinerDrip v1 (mainnet, Oct 2026):** a keyless drip covenant without `tx.inputs.length == 1`. Two equal top-ups could be merged into one output and the whole second coin paid to the miner. Caught in review, fixed in v2, now the first example in the tool's README.
- **A wallet's account contracts (engine, Nov 2026):** 1,924 generated adversarial spends over 54 seed transactions found 7 accepted ones, all one shape: `exit`, `evacuate` and three payee paths did not pin the input count or the input's covenant binding, so a stray coin at the address could be spent by those paths, or a second coin of the same ID burned to fee. Fix: two lines on one contract, one on another.
- **Template ownership (engine):** a contract entry that ignored a token cell it owned let the cell go to an attacker. Fix: refuse to share a transaction with the token.
- **A breached service key paying itself (independent review, Nov 2026):** the same account contracts let the service key alone create an escrow coin with any payee, any expiry and silence resolving to the payee, and its expiry path needed no signature. The adversarial runs did not cover it: their attacker holds only their own key, and the service key is a legitimate signer. Fix: the parent contract fixes the terms of every coin that key can create alone (`04-covenant-design-patterns.md`, principle 14). This is the insider class in section 5, and it was the most important finding of the whole round.

## 5. What the engine and the red team do not cover

Plan separate checks for these.

- **Mempool standardness.** The node's mempool refuses things consensus would accept. Most important: **more than 15 signature opcodes in a P2SH redeem script** (counted without running it) is non-standard and no public node relays it. A contract with 21 passed every engine test and could not be spent on testnet. Count sig ops before deploy (`03-silverscript-v1-gotchas.md` section 6).
- **Storage mass at real values.** Small covenant outputs (below about 0.081 KAS) cannot be created at all. Test with the amounts your users will actually use.
- **Fees in a contended mempool.** Storage-heavy transactions clear the floor but are picked last.
- **Insiders (Tier 2).** The tool's attacker holds only their own key. A party abusing their own legitimate signature (a payee taking more than their share) needs a test written around the roles. For every key in the design, write the sentence "if this key is stolen, the thief can at most ..." and test that bound with that key in the attacker's hands. Service and hot keys first.
- **ZK circuits.** The tool can test everything around a proof (binding outputs to the journal, mempool reuse) but not the circuit or whether the verifying key is right.
- **Off-chain parts.** Watchers, keepers, indexers, the web page, the wallet popup. A perfect covenant behind a page that shows the wrong address loses money.
- **Wallet behaviour.** Sighash types, decoded approval pages, account switches. Test with a hostile page against the wallet itself.

## 6. Unknown programs (no source)

For a covenant you did not write (a token, a pool), the red team's analyzer (`analyzer/` in the same repo) judges on-chain programs by mutating their real spends: it rebuilds an observed spend, replaces the person who made it with an outsider, re-signs, and changes one thing at a time (recipient, amounts, signatures, entry tags, state pushes). If consensus accepts a change that moves holders' value, that is evidence. If nothing gets through, the label stays "unverified": absence of a finding is not a proof, and powers behind a key nobody shares (an issuer's freeze) cannot be witnessed this way.

## 7. What to tell the user

- What was tested: the attack library version, every path explored or not, the constructor values used.
- What passed and what did not.
- What is outside the test (section 5).
- "This is not an audit. It means the attacks we have did not break it."
