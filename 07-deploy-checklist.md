# 07. Deploy checklist

Copy this into the task list of any covenant or wallet project. Every box exists because skipping it cost someone time or money.

## A. Design (before code)

- [ ] Every path written as a plain sentence: who, when, to where, how much fee at most.
- [ ] Every party's "what if they disappear / lose a key / refuse to sign" answered.
- [ ] A path exists that eventually releases the money in every case (no coin stuck forever).
- [ ] "What can a stranger do with this coin?" answered: nothing, or only push it along a fixed path.
- [ ] Network chosen (testnet first). Signing wallet chosen and confirmed able to sign covenant inputs.
- [ ] Compiler version pinned (`silverc` commit). Code written for that version's syntax.

## B. Code

- [ ] Recipients pinned (`scriptPubKey` against constructor values, with the `0000` version prefix where raw bytes are compared).
- [ ] Amounts pinned against `tx.inputs[this.activeInputIndex].value`, with a bounded fee.
- [ ] `require(tx.inputs.length == 1)` wherever the contract assumes it is spent alone.
- [ ] No key, oracle or arbiter taken from the caller.
- [ ] Every hash or secret check paired with a signature or a fully fixed spend.
- [ ] Time locks in the right domain: `this.ageDaa` in DAA (864,000 a day on mainnet), `tx.time` in ms, `date()` in UTC.
- [ ] Fee caps budgeted for the full redeem script size plus change.
- [ ] Signature opcodes counted: at most 15 per P2SH redeem script (each `checkSig` call site counts once, helpers are inlined).
- [ ] Integer math checked for overflow (`value * bps` split into whole and remainder).
- [ ] If the covenant holds or can receive token cells: every entry either refuses to share a transaction with the token or checks its outputs; held cells are borrow-off; cells owned by covenant ID, not by the P2SH of a stateful contract.
- [ ] If the covenant is a lineage: covenant ID minted at funding, binding checked on its own outputs.

## C. Test

- [ ] `kaspa-redteam` with real constructor values: PASSED, 0 findings, every path explored.
- [ ] Spend map read and matches the sentences from A.
- [ ] Contracts that work together tested as one project.
- [ ] Every path spent once on testnet (or with a minimal mainnet amount), with the exact wallet the user will use.
- [ ] Time-locked paths: one spend just before the lock (refused) and one after (accepted).
- [ ] Smallest real amount tested (storage mass), and the largest expected number of coins at the address.

## D. Deploy

- [ ] Address derived from the compiled script yourself and compared with the deploy tool and with the wallet popup.
- [ ] Same rules and same keys as an existing covenant? Then it is the same address. Either deposit there knowingly or change something that actually changes the script.
- [ ] Source, constructor values, compiler commit and the `.ksm` spend map saved by the user, in a place that does not depend on your site.
- [ ] Funding verified on chain (the coin exists at the address with the expected value) before the app shows "deployed".
- [ ] For a vault or anything with a rescue path: rescue address, delay, guardian key and pinned hashes checked from a machine the deploying site does not control, before the large deposit. Rescue address proven by a signature from that wallet.
- [ ] Storage mass of every path computed at the real amounts; at most a few transactions of the heaviest path fit in one block's 500,000 g.
- [ ] Small amount first.

## E. After deploy

- [ ] First real spend of each path watched end to end.
- [ ] Indexers and pages show what the chain shows (balance from the node, not from your database).
- [ ] Users told, in plain words, what the covenant allows and what it does not protect against.
- [ ] Monitoring for anything that should alert a party (an unlock, a pending proposal, an evacuation).

## F. Web apps that touch money

- [ ] Session tokens verified server-side on every money request (JWT signature checked, not just decoded).
- [ ] Payout addresses come from the verified session or the covenant, never from the browser.
- [ ] Account-switch listener and a pre-sign check that the wallet's active address equals the session address.
- [ ] Wallet network checked before signing.
- [ ] Every signature request decoded and shown to the user; SIGHASH_ALL only.
- [ ] Every input the wallet was asked to sign checked for a non-empty signature before broadcast.
- [ ] Static assets versioned (`app.js?v=N`) when the server caches them for long periods, and the CDN confirmed to honour the version (or the file purged). A stale cached script looks exactly like a server bug.
- [ ] Rate limits on build, broadcast and AI endpoints, keyed on the real client IP (`CF-Connecting-IP` behind Cloudflare).
- [ ] Keys, `.env`, API keys and tokens out of git; checked before every commit.
- [ ] Backups of the database off the server.
- [ ] Uploaded files typed from their bytes, never from a `data:` header or file name; served from an allow list of image types with a sandbox CSP.
- [ ] No shared temp paths for per-user files (exports, backups); random per request, deleted after.
- [ ] Every setting that lowers protection (PIN reset, "no password for sends", 2FA off, payee rules) asks for the password or PIN again.
- [ ] Email codes from a CSPRNG, guess-capped per address, send-capped per hour; email verified before any automatic payout or invite claim.
- [ ] OAuth: exact-match redirect URIs, single-use codes consumed atomically, rotating refresh tokens, scopes enforced per call, SameSite cookies, callback URLs https-only with private IPs blocked.
- [ ] Webhooks signed (HMAC, constant-time compare, short timestamp window).
- [ ] Message signing asks the user every time.
- [ ] `localStorage` writes wrapped so a quota error cannot break login; no render-blocking third-party fonts or scripts on the login path.

## G. Services that hold a key (hot wallets, keepers, processors)

Details in `11-hot-keys-signers-watchers.md`.

- [ ] Every signing call site listed; all of them pass one signer gate (CLI tools and cron jobs included).
- [ ] Allow list of transaction shapes, deny by default, pinned by hash; changing it is a deploy.
- [ ] Each signature tied to an intent row created by an authenticated route.
- [ ] Velocity caps with a hold queue; a kill switch read from the database every few seconds.
- [ ] Boot self-test (known signing vector, SDK version against node) refuses to sign on failure.
- [ ] Automatic rules ran in shadow mode against real traffic before enforcing.
- [ ] Money-moving switches need both the database flag and an `.env` line.
- [ ] For every key: "if stolen, the thief can at most ..." written down and tested with that key as the attacker.
- [ ] Watcher on another box, with its own node, its own mail account, immutable registrations, a heartbeat that carries the last successful read, and rescue retries that persist.
- [ ] Drills run on real money: release, "not me" rescue, watcher node down, platform down during a rescue.

## H. Servers

Details in `12-nodes-and-servers.md`.

- [ ] If your server holds a key or runs a watcher or indexer: own node with `--utxoindex`, SDK from the same release, RPC ports firewalled or `--rpcmaxclients` raised. Otherwise public nodes through the `Resolver` and the REST API are enough.
- [ ] One shared RPC connection per process; a timeout on every node call.
- [ ] All boxes on UTC (system, database, process manager).
- [ ] SSH key-only, firewall on, public apps not running as root.
- [ ] Disk: node data dirs, database binlogs and process logs have limits.
- [ ] Backups (database, `.env`, key files, user uploads) encrypted with one offline key, off the box; code pushed to the remote on every deploy.
- [ ] API and signer hostnames not behind a CDN challenge; TLS renewal works behind the proxy.

## I. Questions to ask the user before mainnet

1. "If you lose this key, who can get the money out, and how long does it take?"
2. "If the other party never signs, what happens to the money?"
3. "Is there any path where someone other than the people we named can receive KAS?"
4. "What is the most you are comfortable losing if we are wrong?"
5. "Do you have the source, the values and the spend map saved somewhere other than this site?"
6. "If our server is taken over tonight, what can the attacker do with each key it holds, and who finds out first?"
