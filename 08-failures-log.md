# 08. Failures log

Real mistakes made building covenant tools, wallets, indexers, watchers and token launchers on Kaspa mainnet, 2025 to Nov 2026. Each entry: what happened, why, and the rule that follows. Search this file by symptom before debugging something that looks familiar.

## Consensus and scripts

**The Bitcoin opcode table (cost a night).** A time-locked withdrawal kept failing. The lock reader mapped opcodes by Bitcoin's table (CLTV 0xb1, CSV 0xb2). Kaspa has CLTV at 0xb0 and CSV at 0xb1, so a relative lock was read as an absolute one and the wrong field was set. A whole SDK theory ("the SDK drops lockTime") was built and disproved on the way.
*Rule:* never assume Bitcoin. Read rusty-kaspa source or test on the engine.

**The misleading lock error.** The node said "locktime is greater than the transaction locktime: N > 0". The 0 was the input's sequence, not the transaction's lockTime.
*Rule:* for relative locks, look at the input's sequence first.

**The lock reader missed the compiler's new lock shapes, twice.** silverc v1 inserts range checks between the lock value and the lock opcode (`<N> OP_DUP OP_0 <2^32> OP_WITHIN OP_VERIFY OP_CSV` for relative, `OP_DUP` and a threshold compare for absolute). A reader that looked for "push then opcode" saw no lock, the page said "ready", and the node refused the spend. It happened once for relative locks and again, two days later, for absolute ones.
*Rule:* follow the value through DUP, compare and VERIFY; fail closed on anything unrecognized; test every lock shape the compiler emits, on chain, before calling a lock reader done.

**One path's lock applied to every path.** Reading every lock in the whole redeem script made an escrow's timeout refund gate its normal release too.
*Rule:* scope locks to the branch of the entry being spent.

**The missing selector.** Covenants compiled before silverc v1 select multi-entry paths with a script number after the arguments. A spend that sent `<sig> <redeem>` made the node read the 65-byte signature as the selector: "Number too big … 65 bytes which exceeds the max allowed of 8".
*Rule:* know which compiler made a covenant. v1 contracts take a 4-byte tag, older multi-entry ones take a selector number, older single-entry ones take neither.

**Same address after "changing" the contract.** A user renamed the contract and a parameter to get a fresh covenant. The address did not change: names do not reach the script. Only keys, values, entry names and parameter types do.
*Rule:* detect an existing address at deploy and tell the user.

**Two coins, one output (shipped on mainnet).** MinerDrip v1, a keyless drip, lacked `require(tx.inputs.length == 1)`. Two equal coins at the drip could be spent together against one output and the second coin went to the miner. Caught in review when another project reused it; fixed in v2.
*Rule:* every contract that assumes it is spent alone says so in code. Red-team it (`06-redteam-before-deploy.md`).

**21 signature operations.** A six-contract wallet design passed 1,900+ adversarial engine tests. The first charge on a node was refused: "transaction input #0 has 21 signature operations which is more than the allowed max amount of 15". The mempool rule is policy, not consensus, so the engine never saw it. silverc inlines helpers, so a helper with three `checkSig` branches counted three times per call site.
*Rule:* count sig ops statically before testnet. Select the key, then check once.

**Covenant refused its own spend over the fee.** A partial withdrawal left a small change coin whose storage mass pushed the fee over the covenant's own `MAX_FEE`, so the script refused. The Studio had computed the fee correctly; the covenant's budget was too tight.
*Rule:* size `MAX_FEE` for the worst output shape the path allows, and cap the payout so the change coin is not tiny.

**A unit typed as sompi.** A user deployed with `cap = 4` meaning 4 KAS. The contract read 4 sompi.
*Rule:* read the parameter's role from the source (compared with `.value`, it is an amount) and ask for KAS in the form.

**`date()` is UTC.** The form asked for local time and passed it straight into `date("...")`.
*Rule:* convert or say "UTC" on the form.

**The service key could pay itself.** An account design let the service's hot key alone open an escrow coin with any payee and a "nobody signs" expiry. Thousands of adversarial engine spends passed; an independent cold review found it in one read.
*Rule:* for every key, write what a thief holding it can do, and fix the terms of anything that key creates alone (`04-covenant-design-patterns.md`, principle 14).

**Reading lockTime without CLTV.** On the engine, a contract that read `tx.lockTime` to drive a daily window accepted lockTimes 50 days ahead, because a spender can finalize every input and the node then ignores lockTime. Even with CLTV, lockTime can be backdated.
*Rule:* tie any lockTime read to a CLTV on the same value, and assume backdating.

## Fees and mass

**"Storage mass is the dominant cost" (wrong for a week).** A team believed the relay fee was charged on storage mass and sized every fee budget for it. Reading rusty-kaspa showed the floor applies only to max(compute, transient/2); a 26,095 g storage-mass spend went through on mainnet for 0.00368 KAS.
*Rule:* read the mempool source, then prove the claim with one cheap mainnet transaction.

**The third output.** Passing payout and change as two explicit outputs plus a fee estimate to the SDK made it add a third, tiny change output whose storage mass exceeded the limit.
*Rule:* one payment output, a change address, extra fee as `priorityFee`.

**The Toccata node release raised the floor a hundredfold and one call site kept the old fee.** Every send failed ("transaction has 2069 fees which is under the required amount of 203600"). The legacy wallet SDK defaulted to 1 sompi per gram. A search-and-replace set the new rate before each wallet sync and missed the one sync wrapped in a timeout, which was the main send path.
*Rule:* set fee options where the wallet object is created, then grep that every creation site has it.

**A flat fee stopped working.** A notary service attached 3,000 sompi to every seal; the node started refusing it ("under the required amount … for compute mass") when transactions grew.
*Rule:* compute fees from mass every time.

## Wallets

**Wrong wallet got paid.** A receiver's withdrawal paid the sender. The payout went to the session's address while the extension's active account had changed after login.
*Rule:* drop the session on `accountsChanged`; compare the active address with the session address before every signature.

**Signing on the wrong network.** Kasware was on the IGRA L2 network. The signatures were useless on Kaspa mainnet.
*Rule:* check the wallet's network before every signature request.

**Auto-finalize fails after the user signs.** With `autoFinalized: true`, Kasware could not build the final script for a custom covenant and failed after the signature prompt.
*Rule:* `autoFinalized: false`, assemble the signature script yourself.

**Two-argument `signPskt` crashes.** `signPskt(json, opts)` crashed inside Kasware. Only the single-object form works.

**A silent empty signature.** Kastle's dApp signer returned the transaction with the covenant input's signature script empty and no error.
*Rule:* check every requested input for a signature before broadcast.

**Message signatures changed format.** A wallet update switched `signMessage` from base64 to hex, and a login check that reused another service's endpoint started refusing every extension wallet.
*Rule:* verify message signatures yourself with the SDK and accept the known shapes.

**A token's owner rule read as a script hash.** A wallet hid every holder's KRON tokens because type 3 in KRON's older layout ("the holder's own coin is spent alongside") was read as the reference program's scheme 0x03 ("P2SH owner").
*Rule:* branch on the program layout first, then on the scheme number.

**A passkey that asked for a PIN nobody knew.** A passkey created in desktop Chrome asked, on the phone, for a Google Password Manager PIN the user never knowingly set. The site cannot skip that prompt.
*Rule:* create passkeys on the device in hand, show the recovery-code path from the start, suggest a passkey per device.

**A rescue proof that bound an exchange.** Proving the rescue wallet by "send 0.1 KAS from it" accepted a payment from an exchange, binding the exchange's hot wallet as the rescue address (found in review before any rescue ran).
*Rule:* prove a rescue address by a signature from that wallet.

**A partial signature moved someone's tokens (engine).** A SIGHASH_SINGLE|ANYONECANPAY signature made for an unrelated listing authorized moving all of the signer's KRON older-layout tokens.
*Rule:* SIGHASH_ALL only for anything from outside the wallet.

## SDK and infrastructure

**The process died on disconnect.** `rpc.disconnect()` after a timed-out connect threw outside any try/catch and killed the server during a funding check.
*Rule:* `.catch()` the racing connect promise; skip disconnect when the timeout won.

**gRPC port, WebSocket client.** A client pointed at 16110 (gRPC) hung forever. wRPC Borsh is 17110.

**npm deleted the SDK.** Files placed by hand inside `node_modules` vanished after a routine `npm install`.
*Rule:* vendor the SDK outside `node_modules`.

**Spreading a WASM object copies nothing.** UTXO entries from the SDK are class objects with getters; `{...entry}` was empty. A hand-made entry needs a real `ScriptPublicKey` instance.

**A reconnect loop after the node upgrade.** After the node moved to a new release, the subscription service connected, subscribed, and was dropped on the first notification, forever. The WASM SDK was one release behind; Borsh framing changed. Each reconnect also built a new client without disposing the old one, so "connected" lines stacked up.
*Rule:* upgrade the SDK with the node; dispose the old client before making a new one.

**"Failed to subscribe addresses: undefined" for months.** The call passed `{ addresses: batch }` (the gRPC shape); the WASM SDK wants the array. The SDK throws strings, so the log printed `undefined`.
*Rule:* pass the array; log string errors as strings.

**The node turned the app's own requests away.** Sends hung in "Full wallet sync". The public node had hit its 128-client RPC cap, filled by strangers' wallets and bots. Weeks earlier the same symptom came from the app's own code opening a gRPC connection per send.
*Rule:* `--rpcmaxclients` or a firewall; one shared connection per process.

**A hung sync took every send with it.** The RPC endpoint was a DNS seeder, not a node. One wallet sync never resolved, stayed in a "sync in progress" map, and every later send waited on it until Cloudflare returned 504. The frontend showed "Unexpected token '<'".
*Rule:* timeouts on every node call; point clients at a node you chose; check status codes before parsing JSON.

**A price that was never real.** A price API started refusing the server's IP (403 from its CDN). The code fell back to a hardcoded KAS price, and price history froze for days without an alarm.
*Rule:* never invent a fallback number; use a second real source (api.kaspa.org serves a KAS price) and show "unavailable" when both fail.

**Cloudflare's challenge page in front of an API.** A proxied DNS record for a signing service put a browser challenge in front of every API call.
*Rule:* DNS only for machine-facing hosts.

**Login broken by a font.** During a Google outage, Firefox blocked the page on a Google Fonts stylesheet. Separately, `localStorage.setItem` threw `QuotaExceededError` inside the login flow and showed users the raw message.
*Rule:* no render-blocking third-party assets on the login path; storage writes that never throw.

**Builds failed for a week because of NODE_ENV.** A deploy service ran with `NODE_ENV=production`, which its child `npm install` inherited, silently skipping devDependencies.
*Rule:* strip `NODE_ENV` from build children or install with `--include=dev`.

**The environment variable that did not apply.** `pm2 restart <app> --update-env` with a new variable in front did not reach the process. Deleting and starting the process fresh did.
*Rule:* after changing env, check the running process actually has it.

**A week-long cached script looked like a server bug.** nginx served `.js` with a 7-day immutable cache. A fixed `app.js` never reached browsers and a full debugging round went to a bug already fixed.
*Rule:* bump `?v=` on every asset change and confirm in devtools that the new version loaded. Behind a CDN, check that it honours the query string; if not, purge or rename the file.

**"Try again" re-sent the stale transaction.** After a server fix, the client's retry button re-signed and re-broadcast the old transaction, so the old error came back.
*Rule:* retry means rebuild. Same pre-sign txid means same transaction.

**A stale copy overwrote a live file.** A full file was delivered from a project copy older than production and replaced the live one, breaking every wizard. A delivered config file once crashed a server the same way.
*Rule:* read the live file before writing it; keep backups before every copy; deploy with a script that runs tests and restores on failure.

The same thing happened twice to one core file: the signing function was lost because a full replacement was built from an older copy, restored, then lost again the same way a week later.
*Rule:* before replacing a file, grep the replacement for the functions you know must exist, and build from the server's current copy, never from a project snapshot.

**A test with a fixed date.** A test used a fixed timestamp while the code showed only the last 30 days of facts. It started failing a month later, blocking an unrelated deploy.
*Rule:* tests use times relative to now.

## Indexing

**The wrong covenant for a script.** A column meant "the covenant of the input that revealed this script" actually held the first covenant among the transaction's outputs. A curve showed up as a token cell. Proving each record from its own input fixed 70 rows.
*Rule:* prove relations from the specific input or output, never from "the transaction".

**The wrong creator.** For tokens launched through a launchpad's covenant, the creator was recorded as the launchpad.
*Rule:* when the authorizing input is a covenant, the creator is the first plain-key input.

**Supply undercounted.** Many geneses bind several outputs. Counting only the first, or only spent ones, undercounted supply on two launchpads. One launchpad's second genesis cell was only provable by rebuilding it from a constant its curve program pushes.
*Rule:* prove every bound genesis output; until then, say "at least".

**A false lookalike warning on the real token.** The first lookalike watch flagged the official NACHO for "unsolicited sends": ordinary wallet transfers had been counted as distribution.
*Rule:* combine signals, set thresholds from real data, never warn on the official token for its own name.

**Holder counts that measured airdrops.** Ranking tokens by holders rewarded whoever dusted the most addresses.
*Rule:* count voluntary holders.

**Pinned-only markets missed most trading.** Seeing a market only after pinning its program meant missing every new launcher's pools.
*Rule:* see everything, trust in tiers (`05-kcc20-tokens.md` section 11).

**Icon URLs leaked holders' IPs.** A wallet rendered token icons straight from creator-chosen URLs.
*Rule:* fetch, pin by hash, re-encode and serve icons yourself.

**Dates 55,000 years in the future.** Timestamps from api.kaspa.org are already milliseconds; the code multiplied by 1000.
*Rule:* check the unit of every time field once, in one place.

**A balance that ignored change.** An explorer API's balance for the wallet's first address was shown as the wallet's balance; HD wallets spread coins over change addresses.
*Rule:* sum every derived address, or ask your own wallet service.

**History with holes.** A sync assumed "a stored transaction from time X means everything before X is stored" and fetched only the newest 100. Out-of-order saves made it skip transactions for good.
*Rule:* dedupe by transaction id, page through the whole history.

**Internal transfers shown as payments to strangers.** A transaction whose outputs all went to the user's own addresses was classified as a send with an unknown counterparty.
*Rule:* if every output is the user's own (including their covenant addresses), it is a transfer.

**Token arrivals that never fired.** Live updates subscribed to the holder's address. Token cells never touch it.
*Rule:* learn arrivals from an indexer or the virtual chain (`05-kcc20-tokens.md` section 12).

## Watchers and hosted wallets

**The watcher believed the platform.** Re-registering a vault overwrote the record. A breached platform could have redirected a user's alerts to the attacker, or swapped in a vault the watcher did not know. Found in review; fixed before use.
*Rule:* immutable registrations keyed by covenant address; values derived from the covenant files; the watcher sends its own confirmation email.

**A blind watcher that looked healthy.** The heartbeat said "running" while the watcher could not reach its node.
*Rule:* the heartbeat carries the last successful chain read.

**Then the opposite: a watcher that cried blind.** Slow chain reads (up to 45 s) made the next version page "BLIND" and "recovered" many times a day.
*Rule:* ping the node on its own connection; blind only when the tip cannot be seen.

**A watcher on a public node.** Nine polls in a row failed against a public node that was healthy for everyone else.
*Rule:* give the watcher its own node, with a fallback.

**A rescue that gave up.** One coin's rescue failed, the one-time link was consumed anyway, and no new email went out.
*Rule:* persist the rescue before the first spend; retry until the addresses are empty.

**Daylight saving time paused the alerts.** A box on European time compared a MySQL `DATETIME` with `Date.now()` in the repeated hour; the watcher looked an hour stale and every customer saw "alerts are paused".
*Rule:* UTC everywhere; compute ages in SQL.

**One user could download another user's seed backup.** Exports went to a shared `/tmp/<name>_<date>.zip`.
*Rule:* per-request random paths for anything per-user.

**An avatar that was an HTML page.** The upload's type came from its `data:` header, so `data:image/html` was stored and served from the wallet's own domain (stored XSS).
*Rule:* type files from their bytes; serve an allow list of image types with a sandbox CSP.

**An invite that verified the invitee's email for them.** The old invite flow marked the email verified, and signup only checked that flag.
*Rule:* the person proves the address (link or code) before any claim.

## Process

**Calling it a "smart contract".** The product copy said "smart contracts". Users and even the builder expected things covenants don't do.
*Rule:* say covenants and spend paths.

**A sample from the old compiler.** A `.sil` sample from the testnet era was copied into a new page's generator and failed on the live compiler.
*Rule:* trust a sample only if it says which compiler it compiled on.

**"First multisig on Kaspa."** A launch claim was checked and was false (multisig existed before).
*Rule:* describe what you built; check before claiming "first".
