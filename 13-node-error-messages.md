# 13. Error messages, looked up

Paste the error, find it here, read the fix before changing anything. Every row is a real error from mainnet or testnet work, with the node's or tool's own words where known. Numbers in the messages vary; search for the fixed words.

**If you are an agent:** when a node refuses a transaction, find the message here before you rebuild. If it is not here, read the refusal against rusty-kaspa source before retrying with a different shape (`00-agent-rules.md`, "When to stop and ask").

## Node refusals (mempool and consensus)

| Message (fixed part) | What it means | Fix | See |
|---|---|---|---|
| `transaction input #0 has 21 signature operations which is more than the allowed max amount of 15` (`... is not standard`) | The P2SH redeem script holds more than 15 signature opcodes, counted without running it. Mempool policy; consensus and the script engine accept it, no public node relays it. | Select the key first, check once; merge entries with the same signers. | 03 §6 |
| `transaction storage mass of 800000 is larger than max allowed size of 500000` | An output is too small. Storage mass grows as one over the amount; covenant outputs with a covenant ID count four times. A covenant output under about 0.081 KAS cannot be created. | Raise the smallest output, or drop a tiny change output. | 02 §3 |
| `Storage mass exceeds maximum` (from the SDK while building) | You passed payout and change as explicit outputs plus a fee estimate, and the SDK added a third tiny change output. | One payment output, a change address, extra fee as `priorityFee`. For a full covenant spend, sweep. | 02 §4 |
| `transaction has N fees which is under the required amount of M for compute mass X` | Fee below the relay floor: 100 sompi per gram of max(compute, transient/2) since node v2.0.0 (it was 1 sompi per gram before). Common after the upgrade with old libraries and flat fees. | Compute the fee from mass. With legacy `@kaspa/wallet`, set `wallet.options.minimumRelayTransactionFee` right after creating the wallet (110000 is about 10% over the floor). The required amount in the message is exact; a retry can use it. | 02 §3 |
| `transaction input #0 is not finalized` | The transaction's `lockTime` is not strictly below the virtual DAA score while an input is not final. Setting `lockTime` to "now" causes it. | `lockTime` = now minus a small margin, still past the lock; wait a few blocks if needed. | 02 §4 |
| `failed to verify the signature script: Unsatisfied lock time: transaction input is finalized` | A CLTV path was spent with the input's sequence at MAX. The node then ignores `lockTime`, and the script refuses. | Set the input's sequence below MAX (0 is fine) on CLTV paths. | 02 §4 |
| `locktime requirement not satisfied -- locktime is greater than the transaction locktime: N > 0` | A relative (CSV) lock failed. The `0` is the input's sequence, not the transaction's lockTime. | Set the input's sequence to the lock value before signing. | 03 §3 |
| `sequence locks conditions was not met` | The coin is younger than the relative lock (an early spend), or a coin only a few DAA old was spent at once. The same transaction keeps the same id and is accepted later. | Wait until the coin's DAA score plus the lock is behind the virtual DAA; for fresh coins wait about 20 DAA. | 02 §4 |
| `Number too big ... 65 bytes which exceeds the max allowed of 8` | A pre-v1 multi-entry covenant read your signature as its entry selector. | Push the selector number after the arguments: `<sig> <selector> <redeem>`. | 02 §6, 03 §4 |
| `calculated contextual mass (including storage mass) X is not equal to the committed mass field Y` (node log, transaction inside a block) | Consensus WrongMass. Only applies to transactions arriving inside a block; over RPC the mempool overwrites the field. | Compute the mass with the node's own calculator when you build blocks or hand raw transactions to peers. | 02 §3 |
| A rejected transaction keeps the same id across attempts | You are submitting the same transaction (a fix did not reach the build, or a retry button re-sent the old signed copy). | Rebuild from scratch. | 02 §4 |

## Node and RPC behaviour

| Symptom | What it means | Fix | See |
|---|---|---|---|
| `reached connection capacity of 128` in `journalctl -u kaspad`; app sends stuck in "Full wallet sync" | The node's RPC client cap is full, by strangers or by your own leaked connections. | `--rpcmaxclients=1024` and/or a firewall; one shared connection per process. | 12 §1 |
| `Wallet sync timed out after 30s` | Same as above, or the client points at a seeder instead of a node. | Fix the connection leak; check the RPC endpoint constant. | 12 §1 |
| WebSocket client hangs forever, no error | Pointed at 16110 (gRPC). wRPC Borsh is 17110. | Use the right port. | 02 §2 |
| Connects, subscribes, then disconnects and reconnects in a loop | WASM SDK and node versions differ (Borsh framing is version-coupled). | Use the SDK from the node's release, or the JSON wRPC on 18110. | 12 §1 |
| Several `Connected` lines stacking up after reconnects | Every reconnect created a new `RpcClient` and the old one's listeners stayed bound. | Disconnect, remove listeners and drop the old client before creating a new one. | 02 §2 |
| `Failed to subscribe addresses: undefined` | Two things at once: the WASM SDK's `subscribeUtxosChanged` takes the address array itself, not `{ addresses }`, and the SDK throws strings, so `err.message` is undefined. | Pass the array; log `typeof err === 'string' ? err : err.message`. | 02 §2 |
| UTXO subscriptions silently never fire | Node started without `--utxoindex`. | Restart with `--utxoindex`. | 12 §1 |
| `rpc.disconnect()` crashes the Node process after a connect timeout | The racing connect promise rejected outside your try/catch. | Attach a no-op `.catch()` to it; skip disconnect when the timeout won. | 02 §2 |
| A token arrived but your wallet's UTXO subscription saw nothing | Token cells are P2SH coins that never touch the owner's address. | Learn arrivals from an indexer or from the virtual chain. | 05 §12 |

## SDK and wallet extensions

| Message or symptom | What it means | Fix | See |
|---|---|---|---|
| `... does not contain 'scriptPublicKey' property` | A hand-made UTXO entry used a plain `{version, script}` object. | `new ScriptPublicKey(0, bytes)`. | 02 §2 |
| `{...entry}` of a UTXO entry is empty | SDK entries are WASM class objects with getters. | Read the getters into a plain object yourself. | 02 §2 |
| Kasware: `Cannot read properties of undefined (reading 'length')` on `signPskt` | Two-argument call. | One object: `{ txJsonString, options: { signInputs: [...] } }`. | 02 §5 |
| Kasware fails after the user signed | `autoFinalized: true` on a covenant input. | `autoFinalized: false`; assemble the signature script yourself. | 02 §5 |
| Kasware shows "Balance Change: 0" on a covenant spend | Normal: the input is not the wallet's own coin. | Tell the user in advance. | 01 §3 |
| Kastle returns the transaction with an empty P2SH signature, no error | Its dApp signer did not sign covenant inputs at the last test (Nov 2026). | Check every requested input for a signature before broadcast. | 02 §5 |
| Kaspire rejects at once | The wallet is locked; it does not open an unlock window. | Ask the user to unlock it. | 01 §3 |
| Signatures valid but useless on mainnet | Kasware was on the IGRA L2 network. | Check the wallet's network before every request. | 01 §3 |
| Server-side login check refuses every extension wallet after an update | `signMessage` changed format (base64 to hex; Kaspire returns an object). | Verify with the SDK's `verifyMessage` and accept all known shapes. | 02 §5 |

## Web stack

| Message or symptom | What it means | Fix | See |
|---|---|---|---|
| `Unexpected token '<'` / `... is not valid JSON` on a money call | The reverse proxy (often Cloudflare) returned an HTML 504 because the backend hung. | Find the hang (usually a node call without a timeout); check status codes before parsing JSON. | 12 §3, §4 |
| `The quota has been exceeded` toast at login | `localStorage.setItem` threw `QuotaExceededError`. | A storage helper that falls back and never throws. | 12 §5 |
| Page blank in one browser during a Google outage | Render-blocking Google Fonts stylesheet. | Self-host fonts. | 12 §5 |
| `SSL_ERROR_BAD_CERT_DOMAIN` after moving to Cloudflare | SSL mode Flexible. | Full (strict). | 12 §4 |
| A machine client gets an HTML challenge page from your API | The API hostname is proxied by Cloudflare. | DNS only for API and signer hosts. | 12 §4 |
| A fixed `.js` file keeps behaving like the old one | Long cache at nginx or the CDN; the query string may be ignored. | Purge or rename; confirm the loaded version in devtools. | 12 §4 |
| A new `.env` value is not used | The process never reloaded it, or `dotenv` ran after the module that reads it. | Restart fresh; load `dotenv` first; print the value at startup. | 12 §3 |
| Dates in the year 57,000 | Timestamps from api.kaspa.org (`block_time`) are already milliseconds; something multiplied by 1000. | Do not multiply. | 09 §5 |
| Alerts say "paused" or "stale" for one hour in autumn | A box not on UTC compared a MySQL `DATETIME` during the repeated DST hour. | UTC everywhere; compute ages in SQL. | 11 §6 |
