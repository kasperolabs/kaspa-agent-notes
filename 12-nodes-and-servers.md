# 12. Running nodes and servers for a Kaspa app

The unglamorous part. Every item here took a wallet, a watcher or a payment flow down at least once, Sept 2025 to Nov 2026. rusty-kaspa 2.1.0 unless stated.

## 1. Your own node

- **Run your own node for anything that moves money.** Public endpoints change, rate-limit and go down. Build kaspad from source at a release tag and keep the WASM SDK from the same release (`02-sending-and-signing-kas.md`).
- **`--utxoindex` is required** for `getUtxosByAddresses`, balance queries and `subscribeUtxosChanged`. Without it, address subscriptions do not work.
- RPC listeners: `--rpclisten=<ip>:16110` (gRPC), `--rpclisten-borsh=<ip>:17110` (wRPC Borsh), `--rpclisten-json=<ip>:18110` (wRPC JSON). Bind to `127.0.0.1` for anything only the same box uses.
- **Borsh is version-coupled; JSON is not.** A WASM SDK one release behind the node connects, answers `getInfo`, subscribes, then drops on the first notification it cannot decode, and reconnects forever. Match the SDK to the node, or use the JSON wRPC if you want something that survives node upgrades. (Proven on mainnet at the Toccata upgrade, June 2026.)
- **The node accepts 128 RPC clients by default.** A public node fills that up with strangers' wallets and bots, and then refuses your own app ("reached connection capacity of 128" in `journalctl -u kaspad`). Your app sends then hang in "syncing". Raise it with `--rpcmaxclients=1024`, or firewall the RPC ports to your own boxes. Check with `journalctl -u kaspad --since "-30 min" --no-pager | grep -c "connection capacity"`. (Proven on mainnet, Oct 2026.)
- **Your own app can exhaust it too.** A send path that opened a new gRPC connection per transaction and never closed it filled the node and made every later send time out. Use one shared connection per process, with a timeout on every call.
- **DNS seeders are not RPC servers.** A wallet library defaulted to `seeder1.kaspad.net:16110` for RPC; sends hung. Point every client at a node you chose, in one constant, and check which constant the running code actually reads.
- Decide which node is public and which is private. Keep the node your money paths depend on firewalled to your own boxes; offer a different one to the community if you want to.
- Upgrades: `git fetch --tags`, check out the release tag, `cargo build --release --bin kaspad`, restart the unit, follow `journalctl -u kaspad -f`. Know which binary your systemd unit runs; a stray copy in `/usr/local/bin` is a classic confusion.
- Testnet: testnet-10 is the live Toccata testnet (Nov 2026). Run it as its own unit with its own data dir and ports (for example gRPC 16210, Borsh 17210, JSON 18210, P2P 16211) so it never collides with mainnet.

## 2. Disk

- Old node data directories are huge. A cleanup on one box found about 350 GB of stale node data from replaced units.
- MySQL binary logs grew to 45 GB on an indexer box. Set an expiry (`binlog_expire_logs_seconds`, 2 days is plenty for most) unless you need point-in-time recovery.
- Flush process-manager logs on a schedule. A full disk stops the node, the database and the watcher at the same moment.

## 3. Process manager and environment

- After changing an environment variable, check that the running process has it. `pm2 restart <app> --update-env` did not reach the process once; deleting and starting it fresh did.
- If a module reads `process.env` at require time, `dotenv.config()` must run before that module is first required. A constant that ignores `.env` is usually this.
- A deploy service run with `NODE_ENV=production` passed it to its child `npm install`, which silently skipped devDependencies; every build that needed TypeScript failed for a week. Strip `NODE_ENV` from child processes or use `npm install --include=dev` for builds.
- Wrap every long node call (wallet sync, chain read) in a timeout. A sync promise that never resolved sat in a "sync in progress" map forever, every later send waited on it, and the reverse proxy returned 504 to every user (`13-node-error-messages.md`, "Unexpected token '<'").
- Run public apps as their own users, not root. Keep the signer, key service and watcher on boxes that serve nothing else public.

## 4. Cloudflare and TLS

- **API and signer subdomains: DNS only (grey cloud).** A proxied record put Cloudflare's challenge page in front of an API, and every non-browser client broke. Proxy the website; leave machine-facing hosts unproxied or add rules that skip challenges.
- **SSL mode must be Full (strict)**, not Flexible, when the origin has a certificate. Flexible produced `SSL_ERROR_BAD_CERT_DOMAIN` and redirect loops after moving a domain to Cloudflare.
- **Let's Encrypt HTTP-01 renewal fails behind the proxy.** Switch to the DNS-01 challenge with a Cloudflare API token, or use a Cloudflare origin certificate, before the first renewal date.
- **Cache busting.** Depending on the zone's cache settings, `?v=N` on an asset may not create a new cache entry. If a fixed script does not reach browsers, purge the file or rename it, then confirm in the browser's network tab which version loaded. A stale cached script looks exactly like a server bug.
- Behind Cloudflare, the client IP is in `CF-Connecting-IP`. Include nginx's real-IP config before any IP blocklist or rate limit, and regenerate it when Cloudflare publishes new ranges, or every limit applies to Cloudflare's own addresses.
- When your backend hangs, Cloudflare answers with an HTML 504 page. A frontend that parses every response as JSON then shows "Unexpected token '<'". Check the status code before parsing, and treat this message as "the server hung", not as a frontend bug.

## 5. Frontend dependencies

- **Render-blocking third-party fonts take your login down with them.** During a Google outage, Firefox held the whole page on a Google Fonts stylesheet; mobile with cached fonts worked. Self-host fonts, or load them with the `media="print" onload` pattern.
- **`localStorage.setItem` can throw** (`QuotaExceededError`, private modes, full storage). One uncaught throw in the login flow showed users "The quota has been exceeded" and kept them out with a correct password. Use a helper that falls back to `sessionStorage`, then memory, and never throws. Do not show raw exception text in toasts.

## 6. Servers in general

- SSH by key only; root login and password auth off; a host firewall (ufw or nftables) on every box. A box that will hold a signer or a watcher key goes key-only before the key arrives.
- All boxes in UTC (`11-hot-keys-signers-watchers.md`, section 6).
- Backups: database dumps plus the files you cannot rebuild (`.env`, key files, uploaded user images), encrypted with one key you can store offline, copied off the box. Code comes back from git, so push every deploy; a commit that never reached the remote is not a backup.
- Before a kernel or package update that needs a reboot, list what must come back (process manager apps, systemd units, containers with restart policies) and check them after.
- Keep the node, the app and the watcher on separate boxes, so one provider problem does not take out the alert path together with the thing it watches.
