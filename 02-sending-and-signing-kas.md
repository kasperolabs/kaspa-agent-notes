# 02. Sending and signing KAS (and covenant spends)

Working code for every call in this file is in `14-sdk-cookbook.md`; this file explains the rules behind it.

The mechanics of building, signing and broadcasting Kaspa transactions from code, as of rusty-kaspa 2.1.0 and the Toccata hardfork (active on mainnet since June 30, 2026, DAA score 474,165,565). Everything marked **proven on mainnet** was done with real KAS.

## 1. Units and numbers

- 1 KAS = 100,000,000 sompi. Keep sompi in code, KAS in the UI. The wire format between your own client and server can be KAS strings ("2.5") if you convert in exactly one place.
- The WASM SDK returns balances as `BigInt`. Convert with care (`Number()` loses precision above 2^53 sompi, about 90 million KAS; use BigInt arithmetic for anything that sums).
- DAA score is the chain's clock. Mainnet runs about 10 DAA per second: 864,000 per day.

## 2. The SDK and the node

- **Use the rusty-kaspa wasm32 SDK from the release you run** (the Node.js build in the release zip). The `kaspa` package on npm is version 0.13.0 from 2023 and too old for covenants. Load the SDK by path.
- If you place SDK files by hand inside `node_modules`, a later bare `npm install` can silently delete them. Vendor the SDK outside `node_modules` and load it by path.
- The WASM SDK throws strings, not Error objects. Use `typeof err === 'string' ? err : err.message`.
- **Ports:** wRPC Borsh is 17110 on mainnet. 16110 is gRPC. A WebSocket client pointed at 16110 hangs forever with no error.
- **Disconnect crash:** calling `rpc.disconnect()` on a socket whose connect timed out mid-handshake throws asynchronously, outside your try/catch, and kills the Node process. Attach a no-op `.catch()` to the racing connect promise and skip disconnect when the timeout won.
- UTXO entries from `getUtxosByAddresses` are WASM class objects with getters. Spreading them (`{...entry}`) copies nothing.
- A hand-made UTXO entry must carry a real `ScriptPublicKey` instance (`new ScriptPublicKey(0, bytes)`), not a plain `{version, script}` object, or the SDK says the object "does not contain `scriptPublicKey` property".
- Broadcast through your own node (or a node you choose), not through the wallet's `pushTx`. You get the node's real rejection text instead of a wallet error.
- **`subscribeUtxosChanged` in the WASM SDK takes the address array itself**: `rpc.subscribeUtxosChanged(addresses)`, not `{ addresses }` (that is the gRPC shape). The same goes for `unsubscribeUtxosChanged`. The wrong shape fails with an undefined error message (because the SDK throws strings). Proven on mainnet.
- **Dispose the old `RpcClient` before reconnecting.** A reconnect loop that builds `new RpcClient(...)` each time without disconnecting and removing the old one's listeners stacks event handlers: duplicate "connected" events, overlapping reconnects, events handled twice.
- **Borsh framing is tied to the node version.** An SDK from an older release connects and subscribes, then drops on the first notification it cannot decode. Match versions, or use the JSON wRPC (port 18110 on mainnet, `--rpclisten-json`) where you want tolerance across node upgrades. Node-side details are in `12-nodes-and-servers.md`.
- One shared RPC connection per process, with a timeout on every call. A connection per request leaks until the node's client cap is full (`12-nodes-and-servers.md`).
- **Legacy `@kaspa/wallet` (the old gRPC wallet SDK)** still works for plain sends but predates Toccata: its default `minimumRelayTransactionFee` is 1000 (1 sompi per gram), a hundred times under today's floor. Set `wallet.options.minimumRelayTransactionFee` (110000 gives about 10% over the floor) immediately after `Wallet.fromMnemonic(...)`, on every wallet instance, not around individual sync or send calls: a search-and-replace that inserted it before each `sync()` missed the one call that was wrapped in a timeout, and that was the main send path. Proven on mainnet, June 2026.
- Call `tx.finalize()` on a WASM SDK `Transaction` before reading `tx.id`, and again after you change any field. An id read earlier is not the id the node will report.

## 3. Fees and mass (corrected Oct 2026, measured on mainnet and testnet)

Three kinds of mass exist: compute, transient and storage.

- **The relay fee floor is 100 sompi per gram of max(compute mass, transient mass × 0.5).** Storage mass is not part of the floor. (Source: rusty-kaspa `mining/src/mempool/check_transaction_standard.rs`.) Proven on mainnet: a 3-output spend with 26,095 g of storage mass was accepted at 0.00368 KAS.
- The SDK's `calculateTransactionMass` returns max(compute, storage). Do not use it to compute the fee.
- **Storage mass is a limit, not a fee.** A transaction's storage mass must stay under 500,000 g, and a block's total storage mass under 500,000 g. Under contention the mempool orders by fee over max(compute, storage, transient/2), so storage-heavy transactions are picked last.
- **Storage mass punishes small outputs.** Each output costs about C × p² / amount, with C = 10^12 and p the number of 100-byte storage units the output needs (a P2SH coin with a covenant ID is 2 units, so it costs 4 times a plain output of the same value). On testnet and simnet: a 0.081 KAS covenant output weighs 493,827 g and is accepted; 0.05 KAS gives 800,000 g and is refused ("transaction storage mass of 800000 is larger than max allowed size of 500000"). Small change outputs are the usual cause.
- **You do not need to write the mass field when submitting over RPC.** The mempool computes storage mass itself and overwrites the field (proven on testnet: a deliberately wrong mass was accepted and corrected). The "WrongMass" consensus rule applies to transactions inside a block. If you build blocks or hand raw transactions to peers, compute it with the node's own calculator.
- **The floor rose a hundredfold with the Toccata node release** (rusty-kaspa v2.0.0, June 2026, from 1 to 100 sompi per gram; it is mempool policy, so it applied as soon as a node upgraded, weeks before the hardfork activated). Every library, constant and fee estimate written before then is too low. The node's refusal names the exact required amount ("under the required amount of M for compute mass X"), so a retry can read it.
- **Design for storage mass, not just fees.** It is driven by the smallest outputs. A payment design that created a 0.2 KAS reserve coin and two 0.5 KAS token cells beside one another weighed 340,295 g of storage mass (mass calculator, Nov 2026): two thirds of a block's whole storage budget for one payment, so at most one such payment per block and last in line under contention, while the relay fee was only 0.037 KAS. Fewer, larger covenant outputs per transaction is the fix; raising the cap does nothing.
- **Fixed fees break.** A service that attached a flat 3,000 sompi fee to every transaction started failing when its transactions grew: the node asked for 100 sompi per gram of compute mass. Always compute the fee from mass.
- Covenant scripts can cap their own fee (`MAX_FEE`). Budget for the redeem script you ship: every covenant spend carries its whole redeem script in the signature script, so a 13 KB script costs more to spend than a 1 KB one, on every spend.

## 4. Building a transaction with the SDK

- **One payment output plus a change address.** Passing two explicit outputs (payout and change) plus a fee estimate made `createTransactions` turn the leftover between your estimate and its own fee into a third, tiny change output whose storage mass exceeded the limit ("Storage mass exceeds maximum"). Working shape: one payment output, `changeAddress` set to where change belongs (for a covenant partial withdrawal, the covenant's own address), the extra fee as `priorityFee`, iterate until the fee covers the mass.
- **A full covenant spend is a sweep.** When the whole coin leaves the covenant, do not ask the SDK for a fixed output plus change: the leftover becomes a dust change output and the node refuses it on storage mass. Pass no outputs, set `changeAddress` to the destination and `priorityFee` sized from the signature script, then assert before signing that there is exactly one output, to the expected payee, and that the fee is within the contract's `MAX_FEE`. Proven on mainnet (about 0.0019 KAS per spend for a small contract). `kaspa-ksm`'s `buildSpendTx` helper still builds the fixed-plus-change shape (checked in 1.3.0, Nov 2026); use its `planSpend`, `applyPlan` and `signatureScript` with your own sweep (`14-sdk-cookbook.md` section 8).
- **Exact-amount sends.** Some wallet-library send paths default to "send all" (sweep the wallet into the payment and fee). For a payment, ask for the exact amount with the fee on top, never a sweep, and refuse when the remainder would be too small to stand as an output (0.05 KAS is a safe floor). Several payees in one payment: one transaction with every output paid exactly, fee on top. Proven on mainnet, Oct 2026.
- **Input sequence.** The SDK's default sequence is not 0. Set it explicitly. For a relative time lock (CSV), the input's sequence must equal the lock value. For an absolute lock (CLTV), the transaction's `lockTime` must be set and the input must not be finalized (sequence not MAX).
- Both `sequence` and `lockTime` are in the signature hash. Set them before the wallet signs.
- **lockTime must be strictly below the node's virtual DAA score** (for DAA-domain locks). Setting it to "now" is refused as "transaction input #0 is not finalized". Use now minus a small margin, and wait a few blocks if needed.
- **A coin only a few DAA old** (for example a payout that just landed) can fail the node's sequence-lock check when spent at once. Wait about 20 DAA.
- **sigOpCount per input** must cover every signature check on the path (each `checkSig` or `checkMsgSig` counts 1). It is in the signature hash, so set it before signing.
- **Compute budget per input** (Toccata transactions, version 1): the node refuses an input whose script exceeds its budget. Run the input on the engine locally to learn its script units, then set a budget with margin. It is not covered by the signature hash.
- A Kaspa transaction id does not cover the signature scripts. You know a transaction's id before it is signed, so you can build a second transaction on its unconfirmed output (proven on mainnet: a merge, then a pull built on the merge's output, sent back to back with two wallet prompts).
- If a rejected transaction keeps the same id across attempts, you are submitting the same transaction. A server fix did not reach the build, or a client button re-sent a stale signed copy. Rebuild from scratch.

## 5. Wallet APIs (browser extensions, as of Nov 2026)

| Wallet | Plain payment | Sign a covenant (P2SH) input |
|---|---|---|
| Kasware | `kasware.sendKaspa(address, sompi, {})` | `kasware.signPskt({ txJsonString, options: { signInputs: [{ index, sighashType: 1 }] } })`, **one object argument**. Two-argument forms crash inside the wallet ("Cannot read properties of undefined (reading 'length')"). Proven on mainnet. |
| Kaspire | `kaspire.request({ method: 'sendKaspa', params: { from, to, amountSompi: "<integer string>" } })`, resolves to a txid | Kasware's shape plus a `sender` field: `{ sender, txJsonString, options: { signInputs: [...] } }`. Signs, never broadcasts. Proven on mainnet. |
| Kastle | `kastle.sendKaspa(address, sompi, {})` | Not possible at the last test (Nov 2026): `signTx` returned the transaction with the P2SH input's signature script empty, silently. |

Details that matter:

- `txJsonString` is the rusty-kaspa "safe JSON" of the unsigned transaction. Build it with the SDK (`createTransactions`, then `serializeToSafeJSON`). Do not hand-roll PSKT JSON.
- Leave auto-finalize off. With `autoFinalized: true`, Kasware fails after the user signs, because it cannot build a final signature script for a script it does not know.
- The wallet returns `signatureScript` = `41` + 64-byte Schnorr signature + `01` (the sighash byte). Your code assembles the final signature script (section 6).
- Kasware's serializer may emit Toccata fields (`storageMass`, `covenantId`, `computeBudget`) that an older server-side SDK does not accept. Normalize before deserializing on the server.
- If your own wallet is the one signing for other apps, the signer-side rules (sign exactly what was sent, fill only the requested inputs, check own coins against your node, never broadcast) are in `11-hot-keys-signers-watchers.md` section 4.
- **Message signatures differ by wallet version.** Kasware 0.11.0 `signMessage` returns a 128-character hex signature; older Kasware returned 88-character base64; Kaspire returns `{ address, signature }`. Verify server-side with the SDK's `verifyMessage` against the x-only key in the address, and accept all three shapes.

## 6. Spending a covenant (SilverScript v1)

The signature script for a SilverScript v1 covenant spend is:

```
<entry arguments, in declaration order> <4-byte dispatch tag> <redeem script>
```

- Each argument is a minimal data push. A `sig` argument is the wallet's `41 <sig> 01` push.
- Every v1 contract has a dispatch tag per entry, even single-entry contracts. The tag is derived from the entry's signature (name and parameter types), not from the contract: `spend(sig)` has the same tag in any contract.
- Covenants compiled before silverc v1 have no tag. Multi-entry ones select the entry with a script number (`OP_0`, `OP_1`, … in source order) after the arguments: `<sig> <selector> <redeem>`. Sending `<sig> <redeem>` to such a script makes the node read the signature as the selector: "Number too big … 65 bytes which exceeds the max allowed of 8". See `03-silverscript-v1-gotchas.md`.
- Each P2SH input needs its own signature and its own redeem script push. Every UTXO at a covenant address is spent independently; there is no "balance", only coins.
- Read time locks from the redeem script bytes (or the compiled artifact), not from source text, and set sequence or lockTime accordingly. Scope the lock to the branch of the entry you are spending, or one path's timeout will be applied to every path.
- Before asking the user to sign, run the spend through the script engine (or at least dry-run the rules) and refuse with a plain message when the rules would refuse it. "Ready" on a page followed by a node rejection is the worst experience.

## 7. Server-side identity

- Verify session tokens (JWT) with the secret or the issuer's verify endpoint on every request that touches money. Decoding is not verifying. A server that only decoded tokens accepted forged sessions until fixed.
- The payout address is the verified session's address or a party key from the covenant itself. Never an address sent by the browser.
- Rate-limit endpoints that build or broadcast transactions.

## 8. Many coins at one address

- A covenant address can hold many coins (every deposit is a coin). Users think in balances. Build a sequencer: if one coin covers the request, spend it alone; if not, merge first and then pull. Above two coins, merging is cheaper than per-coin pulls.
- A heavily funded address made of many tiny coins can hold dust that costs more to spend than it is worth.
- A covenant that allows a partial withdrawal with change back to itself restarts any relative time lock on the change coin. Warn the user before a partial pull on a relative-lock path.
