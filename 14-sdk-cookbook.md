# 14. SDK cookbook: the calls that actually work

Working call shapes for the rusty-kaspa WASM SDK (v2.1.0, Node.js build) and the `kaspa-ksm` covenant helper, taken from code that sends, signs and spends covenants on mainnet. Copy the shapes, keep the checks. When your SDK version differs, read its `kaspa.d.ts` before trusting any line here.

Everything in this file is **proven on mainnet** unless marked otherwise.

## 1. Getting the SDK

- Download `kaspa-wasm32-sdk-v<version>.zip` from the rusty-kaspa GitHub release that matches your node (`kaspad --version`). Never mix versions: the Borsh RPC framing changes between releases.
- Use the `nodejs/kaspa` folder in Node.js and the `web/kaspa` folder in browsers. Put it in your repo outside `node_modules` (for example `vendor/kaspa-wasm/`) and `require` it by path. A later `npm install` deletes anything you put inside `node_modules` by hand.
- The npm package named `kaspa` is a 2023 build (0.13.0) without covenant support. Do not use it for new work. Check any other npm package claiming to be the SDK before installing it; typosquats exist on every chain.
- In Node.js the SDK's RPC client needs a W3C WebSocket. Install the `websocket` npm package and set it once, before the first `RpcClient`:

```js
globalThis.WebSocket = require('websocket').w3cwebsocket;
const kaspa = require('./vendor/kaspa-wasm/nodejs/kaspa');
```

- The SDK throws plain strings, not `Error` objects. Wrap every catch:

```js
const errMsg = e => (typeof e === 'string' ? e : (e && e.message) || String(e));
```

- The zip contains `kaspa.d.ts`. When you are unsure of a function's name or argument shape, read it there. Do not guess from Bitcoin libraries or older Kaspa SDKs.

## 2. The pieces you need

| Need | SDK piece |
|---|---|
| Talk to a node | `RpcClient`, `Encoding` |
| A key to sign with | `PrivateKey` (from 32-byte hex) |
| Build and pay a normal KAS send | `createTransactions` (returns pending transactions with `.sign()` and `.submit()`) |
| Load, inspect, sign and serialize a transaction someone else built | `Transaction.deserializeFromSafeJSON`, `tx.serializeToSafeJSON()`, `tx.finalize()`, `tx.id`, `tx.inputs` |
| Sign one input with SIGHASH_ALL | `createInputSignature`, `SighashType.All` |
| Turn a script into an address | `ScriptPublicKey`, `addressFromScriptPublicKey` |
| Covenant files, spend plans, signature scripts | `kaspa-ksm` (npm): `verify`, `planSpend`, `applyPlan`, `signatureScript` / `finalize` |
| Compile SilverScript | `silverc` binary, pinned to one commit (`03-silverscript-v1-gotchas.md`) |

That is the whole list for sending, signing, covenant spends and reading coins. Anything else, look up in `kaspa.d.ts` first.

## 3. Connecting to a node

```js
const NETWORK = 'mainnet';                      // or 'testnet-10'
const NODE = process.env.KASPA_BORSH || null;   // your own node, e.g. 'ws://127.0.0.1:17110'; unset = public nodes

function withTimeout(p, ms, label) {
  let t;
  return Promise.race([p, new Promise((_, rej) => { t = setTimeout(() => rej(new Error(`${label} timed out`)), ms); })])
    .finally(() => clearTimeout(t));
}

async function withRpc(fn) {
  const rpc = NODE
    ? new kaspa.RpcClient({ url: NODE, encoding: kaspa.Encoding.Borsh, networkId: NETWORK })   // your own node
    : new kaspa.RpcClient({ resolver: new kaspa.Resolver(), networkId: NETWORK });               // a public node
  await withTimeout(rpc.connect(), 20000, 'node connect');
  try {
    const info = await rpc.getInfo();
    if (!info.isSynced) throw new Error('node not synced');
    if (!info.isUtxoIndexed) throw new Error('node started without --utxoindex');
    return await fn(rpc);
  } finally {
    try { await rpc.disconnect(); } catch (_) { /* already closed */ }
  }
}
```

- Without `KASPA_BORSH` set, the SDK's `Resolver` picks a public node (the pattern of rusty-kaspa's own examples). That is enough for apps whose server holds no key (`00-agent-rules.md`, "How strict to be"). Set your own node when the server signs or watches.
- 17110 is wRPC Borsh, 16110 is gRPC. A WebSocket client pointed at 16110 hangs with no error.
- A short-lived client per operation (as above) is fine for occasional sends. A busy service keeps one long-lived client per process instead (section 9). Never one per request without a `disconnect()`; that fills the node's client cap.
- Put a timeout on every call. A call that never returns will hold up everything queued behind it.

## 4. Reading coins and the chain clock

```js
// Unspent coins at one or more addresses. Note: an object with `addresses` here.
const { entries } = await rpc.getUtxosByAddresses({ addresses: [address] });

// Field paths differ slightly between builds; normalize once.
function coinOf(e) {
  const u = (e && (e.utxoEntry || e.entry)) || e;
  const op = (e && e.outpoint) || u.outpoint;
  return {
    txid: String(op.transactionId).toLowerCase(),
    vout: Number(op.index),
    amount: BigInt(u.amount),                 // sompi, BigInt
    daa: BigInt(u.blockDaaScore),             // DAA score of the block that accepted it
    covenantId: u.covenantId ? String(u.covenantId) : null,
    raw: e,                                   // pass this unchanged to createTransactions
  };
}

// The chain's clock, for lockTime and relative locks
const { virtualDaaScore } = await rpc.getBlockDagInfo();
```

- Amounts are BigInt sompi. Never convert to `Number` before summing.
- A coin at a plain `kaspa:q` address that carries a `covenantId` is not plain KAS (it is a token cell or similar). Do not spend it as KAS.
- Keep `raw` and hand it to `createTransactions` as is. A hand-made entry needs a real `new kaspa.ScriptPublicKey(0, hex)`, not a `{version, script}` object.

## 5. Sending KAS from a key you hold

```js
async function sendKas(rpc, privKeyHex, fromAddress, payments /* [{address, amount: BigInt}] */) {
  const { entries } = await rpc.getUtxosByAddresses({ addresses: [fromAddress] });
  const total = payments.reduce((a, p) => a + p.amount, 0n);
  const available = entries.reduce((a, e) => a + BigInt(coinOf(e).amount), 0n);
  if (available - total < 5_000_000n) throw new Error('keep at least 0.05 KAS for the fee');

  const { transactions, summary } = await kaspa.createTransactions({
    entries,
    outputs: payments,                  // every output paid exactly
    changeAddress: fromAddress,         // change comes back to the sender
    priorityFee: 0n,                    // the SDK adds the mass-based fee itself
    networkId: NETWORK,
  });

  const key = new kaspa.PrivateKey(privKeyHex);
  const submitted = [];
  try {
    for (const pending of transactions) {       // may be several: consolidations first, payment last
      await pending.sign([key]);
      submitted.push(String(await pending.submit(rpc)));
    }
  } catch (e) {
    // Something may already be on the network. Report it; never retry blind.
    e.submitted = submitted;
    throw e;
  }
  return { txid: String(summary.finalTransactionId || submitted.at(-1)), txids: submitted, feeSompi: BigInt(summary.fees || 0) };
}
```

- Before signing with a key you derived from a seed, derive its address and compare it with the address you expect to spend from. Refuse on a mismatch.
- If the user wants "send everything", that is a different call shape (no outputs, `changeAddress` = destination). Never let a normal payment fall into a sweep.
- Map node errors to plain words: "storage mass" means an amount is too small; "insufficient" means not enough to cover amount plus fee. Full list: `13-node-error-messages.md`.

## 6. Signing a transaction someone else built (PSKT / SafeJSON)

The format browser wallets and apps exchange is the SDK's "safe JSON" of a transaction, plus a list of `{ index, sighashType }` to sign.

```js
function signInputs(txJsonString, privKeyHex, indexes) {
  const tx = kaspa.Transaction.deserializeFromSafeJSON(txJsonString);   // throws if it is not a transaction
  const key = new kaspa.PrivateKey(privKeyHex);
  for (const i of indexes) {
    const sig = String(kaspa.createInputSignature(tx, i, key, kaspa.SighashType.All)).toLowerCase();
    if (!/^41[0-9a-f]{128}01$/.test(sig)) throw new Error(`unexpected signature shape on input ${i}`);
    tx.inputs[i].signatureScript = sig;      // a push of the 64-byte Schnorr signature plus the sighash byte
  }
  const signed = tx.serializeToSafeJSON();

  // Only signature scripts may differ from what you were given
  const a = JSON.parse(txJsonString), b = JSON.parse(signed);
  if (a.inputs.length !== b.inputs.length || a.outputs.length !== b.outputs.length ||
      String(a.lockTime) !== String(b.lockTime) ||
      a.outputs.some((o, n) => String(o.value) !== String(b.outputs[n].value))) {
    throw new Error('transaction changed during signing');
  }
  return signed;
}
```

- `SighashType.All` only, for anything that came from outside your own code (`01-wallet-safety.md`).
- For a P2PK input, `41<sig>01` is the complete signature script. For a covenant input it is one argument; the app adds the other arguments, the dispatch tag and the redeem script (section 8).
- Before signing coins the app says are the user's, look each outpoint up on a node you trust (section 4) and compare amount and script. Refuse if anything differs.
- If you sign for other apps, sign and return. Do not broadcast for them.

## 7. Broadcasting a finished transaction

```js
async function broadcast(rpc, txJsonString) {
  const tx = kaspa.Transaction.deserializeFromSafeJSON(txJsonString);
  tx.finalize();                                   // the id is only right after finalize()
  const expected = String(tx.id).toLowerCase();
  try {
    const res = await rpc.submitTransaction({ transaction: tx, allowOrphan: false });
    return String((res && res.transactionId) || expected).toLowerCase();
  } catch (e) {
    const msg = errMsg(e);
    if (msg.includes(expected) && /already in the mempool|already accepted/i.test(msg)) return expected;  // a retry of a success
    throw new Error(msg);
  }
}
```

- The transaction id does not cover signature scripts, so you know it before signing. If a rejected transaction keeps the same id on every retry, you are sending the same transaction; rebuild.
- `allowOrphan: false` unless you deliberately build on an output that is not yet accepted.

## 8. Spending a covenant coin (one coin in, one output out)

The shape that works on mainnet: plan the path with `kaspa-ksm`, build a **sweep** with the SDK, check it, write the lock fields, sign, assemble the signature script, broadcast.

```js
const ksm = require('kaspa-ksm');

async function spendCovenantCoin(rpc, { manifest, entry, args, coin, destination, keys /* {sigParam: privHex} */, maxFeeSompi }) {
  const check = ksm.verify(manifest);
  if (!check.ok) throw new Error('covenant file does not verify: ' + check.errors.join('; '));

  const { virtualDaaScore } = await rpc.getBlockDagInfo();
  const plan = ksm.planSpend(manifest, entry, {
    args: args || {},
    chain: { daa: virtualDaaScore, nowMs: Date.now() },
    utxos: [{ blockDaaScore: coin.daa.toString() }],
  });
  if (!plan.ready) throw new Error('not spendable yet: ' + plan.blockers.join('; '));
  const dest = plan.payTo ? plan.payTo.address : destination;    // a path that pins its payee wins

  // The SDK sizes the fee without the signature script it cannot see; add it as priority fee
  const sigScriptBytes = plan.layout.reduce((n, x) => n + (x.kind === 'sig' ? 66 : x.hex.length / 2), 0)
                       + plan.suffixHex.length / 2;
  let priority = (BigInt(sigScriptBytes) + 64n) * 100n;
  if (priority < 20000n) priority = 20000n;

  // Sweep: no outputs, change goes to the destination
  const settings = { entries: [coin.raw], outputs: [], changeAddress: dest, priorityFee: priority, networkId: NETWORK };
  let built;
  try { built = await kaspa.createTransactions(settings); }
  catch (_) { delete settings.outputs; built = await kaspa.createTransactions(settings); }   // some builds want it absent
  if (built.transactions.length !== 1) throw new Error('expected exactly one transaction');
  const unsignedJson = built.transactions[0].serializeToSafeJSON();

  // Check before anyone signs: one in, one out, to the right address, fee within the covenant's cap
  const raw = JSON.parse(unsignedJson);
  if (raw.inputs.length !== 1 || raw.outputs.length !== 1) throw new Error('not a one-in-one-out spend');
  const spk = raw.outputs[0].scriptPublicKey;
  const outHex = String(typeof spk === 'string' ? spk : spk.script).toLowerCase().replace(/^0000/, '');
  const outAddr = kaspa.addressFromScriptPublicKey(new kaspa.ScriptPublicKey(0, outHex), NETWORK).toString();
  if (outAddr !== dest) throw new Error(`output pays ${outAddr}, expected ${dest}`);
  const fee = coin.amount - BigInt(raw.outputs[0].value);
  if (fee <= 0n || fee > maxFeeSompi) throw new Error(`fee ${fee} outside the covenant's cap`);

  // sequence, lockTime and sigOpCount go in BEFORE signing; they are all in the signature hash
  const txJson = ksm.applyPlan(plan, unsignedJson);

  const tx = kaspa.Transaction.deserializeFromSafeJSON(txJson);
  const sigs = {};
  for (const name of plan.signaturesNeeded) {
    const s = String(kaspa.createInputSignature(tx, 0, new kaspa.PrivateKey(keys[name]), kaspa.SighashType.All)).toLowerCase();
    if (!/^41[0-9a-f]{128}01$/.test(s)) throw new Error(`unexpected signature shape for ${name}`);
    sigs[name] = s;
  }
  const signed = JSON.parse(txJson);
  signed.inputs[0].signatureScript = ksm.signatureScript(plan, sigs);   // <args> <dispatch tag> <redeem script>
  return broadcast(rpc, JSON.stringify(signed));
}
```

- **Why a sweep:** asking the SDK for a fixed output plus change on a covenant spend leaves the difference between your fee estimate and the SDK's as a tiny change output, and the node refuses it on storage mass. `kaspa-ksm`'s own `buildSpendTx` helper (checked in 1.3.0, Nov 2026) builds that fixed-output-plus-change shape; use `planSpend`, `applyPlan` and `signatureScript` from its core with your own sweep, as above.
- Measured: a small single-signature covenant spend costs about 0.0019 KAS this way.
- **Exact amounts from a sweep** (an invite that must deliver exactly 5 KAS): build once, read the SDK's base fee (`fee - priority`), rebuild with `priority = (coin.amount - wanted) - baseFee` so the single output lands on the wanted value. The surplus becomes fee, never a second output. Rebuild once more if storage mass moved the base fee.
- **Partial withdrawals** (part leaves, the rest stays in the covenant): one payment output, `changeAddress` = the covenant's own address, extra fee as `priorityFee`. The change coin's relative age starts again at zero.
- When the user signs in a browser wallet instead of your server, send `txJson` (after `applyPlan`) to the wallet's `signPskt`, take the `41<sig>01` it returns for each input, and assemble the same way (`02-sending-and-signing-kas.md` section 5 has the wallet calls).
- An early spend of a relative lock is refused with "sequence locks conditions was not met". The same transaction, unchanged, is accepted once the coin is old enough.

## 9. Following addresses live

```js
class Follower {
  async connect(addresses) {
    if (this.rpc) {                                   // dispose the old client, or listeners stack up
      try { await this.rpc.disconnect(); } catch (_) {}
      try { this.rpc.removeAllListeners?.(); } catch (_) {}
      this.rpc = null;
    }
    const rpc = this.rpc = NODE
      ? new kaspa.RpcClient({ url: NODE, encoding: kaspa.Encoding.Borsh, networkId: NETWORK })
      : new kaspa.RpcClient({ resolver: new kaspa.Resolver(), networkId: NETWORK });

    // Register listeners BEFORE connecting and subscribing. Payloads arrive under event.data.
    rpc.addEventListener('connect', () => { this.connected = true; });
    rpc.addEventListener('disconnect', () => { this.connected = false; this.scheduleReconnect(addresses); });
    rpc.addEventListener('utxos-changed', ev => this.onUtxos(ev.data));            // { added: [...], removed: [...] }
    rpc.addEventListener('virtual-daa-score-changed', ev => { this.daa = BigInt(ev.data.virtualDaaScore); });

    await rpc.connect();
    const info = await rpc.getInfo();
    if (!info.isUtxoIndexed) throw new Error('node needs --utxoindex');
    await rpc.subscribeVirtualDaaScoreChanged();
    await rpc.subscribeUtxosChanged(addresses);       // the array itself, not { addresses }
  }
}
```

- One payment fires several `utxos-changed` events in a burst. Serialize balance updates per wallet, or you will report the same receipt twice.
- Re-read the balance from the node after events; do not add up event deltas forever.
- Public explorer APIs lag the node push. If you also fetch history from one after an event, retry for a few seconds before concluding nothing arrived.
- Token arrivals do not show up here (`05-kcc20-tokens.md` section 12).
- Reconnect with backoff, and re-subscribe every address after each reconnect.

## 10. Units and addresses

```js
const SOMPI = 100_000_000n;

function kasToSompi(text) {                         // "2.5" -> 250000000n, exactly
  const m = /^(\d+)(?:\.(\d{1,8}))?$/.exec(String(text).trim());
  if (!m) throw new Error('amount must be KAS with up to 8 decimals');
  return BigInt(m[1]) * SOMPI + BigInt((m[2] || '').padEnd(8, '0'));
}

function sompiToKas(sompi) {                        // 250000000n -> "2.50000000"
  const s = BigInt(sompi);
  return `${s / SOMPI}.${(s % SOMPI).toString().padStart(8, '0')}`;
}

// Address from a script, e.g. an output's scriptPublicKey (strip a leading 0000 version if present)
const addr = kaspa.addressFromScriptPublicKey(new kaspa.ScriptPublicKey(0, scriptHex), NETWORK).toString();
```

- Convert between KAS and sompi in exactly one place. Users type KAS; scripts and the SDK take sompi.
- Script shapes you will meet: P2PK Schnorr `20<32-byte key>ac` (`kaspa:q…`), P2PK ECDSA `21<33-byte key>ab`, P2SH `aa20<32-byte hash>87` (`kaspa:p…`). Inside a covenant script they carry a `0000` version prefix.
- Do not write your own bech32 for Kaspa addresses; use the SDK.

## 11. Legacy `@kaspa/wallet`

Older code derives keys and sends with the gRPC wallet SDK `@kaspa/wallet`. It still works, but:

- Set `wallet.options.minimumRelayTransactionFee = 110000` right after `Wallet.fromMnemonic(...)` on every wallet instance. Its default predates the current fee floor by a factor of 100.
- Reuse one gRPC connection per process and close what you open.
- For anything with covenants or the newer transaction fields, use the WASM SDK. Keys derived with `@kaspa/wallet` can sign through the WASM SDK as `new kaspa.PrivateKey(hex)`; check the derived address first.
