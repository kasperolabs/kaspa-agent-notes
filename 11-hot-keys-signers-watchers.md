# 11. Services that hold a key: hot wallets, signers, watchers

Most of this library is written from the dApp side: a user's wallet signs, your page builds. This file is for the other side: a server that holds a key and signs by itself (a hosted wallet, a payment processor, a keeper, a launchpad, a bot), and the watcher that keeps an eye on it. Learned on mainnet from hosted wallets, vaults with outside watchers and payment rails, Sept to Nov 2026.

Status labels as elsewhere: **proven on mainnet**, **proven on testnet**, **engine-tested**, **read in source**. Lines without a label are practice, not chain facts.

## 1. What actually protects a hot key (less than you think)

- A server that signs by itself must be able to decrypt its keys without a human. So the key that decrypts them sits on the same box, usually in `.env`. Whoever gets the database and `.env` gets every key. A user's account password does not protect their key on a taken box.
- The dangerous attacker does not need to steal the key. If they can run code where your signer runs, they can call your own signing function and it will sign for them. The largest hot-wallet theft on record (Binance, Oct 2026, section 10) went through the exchange's signing service, not a stolen key. Defend the signing call, not only the key file.
- A DB-only leak (a stolen dump or backup without `.env`) should yield nothing. Keep the decryption key out of the database and out of the backups that leave the box unencrypted.
- Hot keys belong on a box that serves no public web app. If the signer, a key service or a watcher must be reachable, give it its own small box and its own domain.
- Code that touches a customer-held key (a key page, an enrollment page) must not be served by the server that holds the hot key. A breached app server would otherwise serve a page that copies the customer key on the first tap. Serve it static and hash-pinned from another box. (Found by an outside review of a design, Nov 2026.)

## 2. A signer gate: one door for every signature

Build one module that every signature in the service passes through. No code path may reach the SDK's signing function directly. A design that has run in production (shadow mode first, Oct 2026):

1. **Inventory first.** List every place the service signs: plain sends, multi-output payments, token sends and merges, invite locks, vault paths, external signing requests, payload transactions, cron jobs, CLI tools. Missing one call site defeats the gate. CLI tools are not exempt.
2. **Deny by default.** A pinned allow list of transaction shapes, one entry per purpose: allowed input script types (own P2PK, or P2SH whose script hash is on a pinned list), allowed outputs, SIGHASH_ALL only, payload rules, maximum inputs. Anything that matches no entry is refused and logged as a red event. Changing the list is a deploy, never a database edit.
3. **Intent binding.** Every signature carries an intent id. An intent row is created only by an authenticated user route (or a named job like an invite return) and records who, purpose, destinations and amounts. The gate recomputes outputs from the real transaction and compares. An attacker who can call the signer but cannot forge an intent through the normal route gets nothing.
4. **Velocity caps with a hold queue.** Platform-wide outgoing value per hour and per day, plus a per-account day cap. Above the cap, transactions wait for a human instead of failing. Set caps well above real peaks; a hold is a delay, never a loss.
5. **Boot self-test.** On start, sign a known vector, compare address encodings between two libraries, check the SDK version against the node. Any failure: refuse to sign. This catches a swapped or tampered SDK before the first real signature.
6. **A kill switch read every few seconds from the database**, not from `.env`, so it works without a restart and can be flipped by an admin page or by the watcher. Keep an env override too. The kill switch check must sit outside any try/catch so it holds when the rest of the gate is broken.
7. **Shadow mode first.** Run every automatic rule in log-only mode ("would have refused") for a week or more, against real traffic, before enforcing. Every "would refuse" is either a missing allow-list entry or a real finding.
8. **The config is part of the attack surface.** The allow list, thresholds and emergency addresses are pinned by hash and watched. Someone else changing them should page you.
9. **Silence is a signal.** A wire that stops reporting is treated as tripped.

## 3. Switches that can move money

- New money features ship behind a switch, off by default.
- A switch that can move money or change what a signer signs needs two keys: the database row and a matching line in `.env`. A stolen admin session alone then cannot turn it on. People find this annoying. It is deliberate.
- Message signing is a key use. An endpoint that signs any message without the user's approval can sign another site's login challenge in the user's name (found in a review of a live hosted wallet, Oct 2026). Gate it like a send.

## 4. Approvals in a hosted wallet

- Every external signing request and every app payment is approved by the user on the wallet's own origin, each time, unless the user created a capped, revocable mandate inside the wallet itself.
- Sign exactly the transaction you were given. Change nothing (version 1 fields such as `computeBudget`, the inputs' `covenantId` and the outputs' covenant bindings are all in the signature hash). Fill the signature only on the input indexes the request named. Never broadcast on the app's behalf.
- Before signing any of the user's own coins for an app, look each one up on your own node: unspent, same amount, same script. If the app's copy differs, refuse.
- When the user's own token coins are among the inputs, name them first on the approval page ("This moves your 100 TOKEN"). Apps add token inputs silently; see the SIGHASH warning in `01-wallet-safety.md`.
- A coin the app just created (a buy's new token coin, a pool successor) may be seconds old. Allow a short retry window when looking it up, and do not demand confirmations for P2SH inputs the app built on its own unconfirmed output.

## 5. Customer keys, passkeys and PINs

If you let customers hold their own key inside your web app (the safer design), these are the traps:

- **WebAuthn PRF gives a different secret per credential and per password store.** One user can have several passkeys for your domain (Chrome, Windows Hello, a phone). Bind the customer key to one credential id, pass it in `allowCredentials` on every unlock, and refuse a different passkey even if it is a valid one for your site. Never "whatever passkey the user taps". (Proven on real devices, Nov 2026.)
- **A passkey made in desktop Chrome on Windows or Linux is encrypted with a Google Password Manager PIN.** The first time a phone uses that synced passkey, Google asks for that 6-digit PIN, which most people never knew they set. Your site cannot skip it. Create the passkey on the device the customer is holding, treat sync as a bonus, show "use a recovery code instead" from the start, and after a recovery-code sign-in suggest adding a passkey for that device.
- Do not add an email fallback for a second factor. It makes the second step exactly as strong as the inbox it was meant to protect against.
- A recovery code or password that appeared in a chat, a ticket or a log is spent. Make new ones.
- Recovery that lets a new key take over an account needs a waiting period the old key can cancel (7 days is a reasonable default), and the customer must hear about it on a channel the attacker does not control.

## 6. Watchers

A watcher is a service on another box that follows the chain, alerts people, and can run an emergency path (rescue, evacuate) with its own key. It is the part of a vault design that fails quietly. Every rule below comes from a real incident or a review of a production watcher.

**Trust**
- **The watcher must not believe the platform.** A first-version watcher overwrote a registration whenever the platform re-sent it. A breached platform could have pointed a user's alerts at the attacker's email, or registered a fresh vault pair so the real one went unwatched. Fix: key registrations by covenant address, make them immutable (a second registration must be identical or it is refused with 409 and the operator is told), and derive every value (airlock address, rescue address, delay, keys) from the covenant files themselves, never from fields the platform sends. (Proven on mainnet: the attacker-email re-registration got 409.)
- **The watcher sends its own "your vault is being watched" email** from its own box and its own mail account at registration. No watcher email means not watched. It is the one check a hacked platform cannot fake. Use a mail account the platform does not share, or a breached platform can send "false alarm, ignore" as the watcher.
- A hacked watcher box can evacuate every vault to its rescue wallet. That is not theft, but a user who lost the rescue seed loses the money. Warn loudly about the rescue seed at setup.

**Sight**
- **A blind watcher looks healthy.** A heartbeat that says only "I am running" hides a watcher whose node is unreachable. Put the time of the last successful chain read in the heartbeat, and count a stale read as silence.
- **A slow read is not blindness.** Full virtual-chain reads (`getVirtualChainFromBlockV2` with transactions) sometimes take 45 seconds on a busy box. A watcher that called itself blind after 15 seconds without a finished read paged its operator twice a spell, many times a day. Ping the node on a second connection every few seconds (server info, DAG info, synced flag, tip). Call it blind only when the tip cannot be seen: the node silent for 60 s, unsynced for 60 s, its chain not moving for 90 s, the follower more than about 1,200 DAA behind the tip, or no read at all for a few minutes. (Testnet watcher, Nov 2026.)
- **Give the watcher its own node.** A watcher reading a public node went blind nine polls in a row while that node was healthy for everyone else. Read a node on the same box first, a second node as fallback, and keep one connection open instead of reconnecting per poll.
- Several watchers with the same key on different boxes, each able to catch up from the platform by itself, so one survivor keeps alerts and rescues going.

**Acting**
- **Persist a rescue request before the first spend, and retry until the addresses are empty.** A first-version rescue gave up on a coin that failed, consumed the one-time link anyway, and sent no new email. A partial rescue must keep going by itself and say how many coins are still to come.
- Rescue must not depend on the platform being up. Reports back to the platform are fire-and-forget.
- An intentional upgrade on the platform must never trigger mass evacuations. Announce upgrades to the watcher from the operator's side, and tell customers by email what changes and why.
- Give emergency paths (rescue, evacuate) a higher fee cap than normal paths. In a fee spike the attacker and the rescuer are capped alike, but only the rescuer is on a clock.
- Run drills on your own money: let a release land, press "not me", stop the watcher's node, take the platform down during a rescue. Write down the time from alert to rescue.

**Clocks**
- **Run every box in UTC** (system, MySQL `time_zone='+00:00'`, `TZ=UTC` for the process manager). A box on European time compared a MySQL `DATETIME` with `Date.now()` during the repeated hour when clocks fell back; the heartbeat looked an hour stale and every customer saw "alerts are paused". Compute ages inside SQL (`TIMESTAMPDIFF(SECOND, t, NOW())`) instead of converting in JavaScript. Print times in emails with the zone named.

## 7. Tripwires for a hosted wallet

Cheap compared with a loss; some of these have run in production:

- **Pre-announce and unexplained spends.** After signing and before broadcast, the signer tells the watcher which outpoints it is about to spend. The watcher subscribes to every custodial address. A spend it was not told about means keys are being used outside your code: alarm, and arm the emergency paths.
- **Canaries.** Small funded wallets whose seeds sit in plausibly named files around the server, with their addresses on the watcher's alert list. A drain tells you what was read and when.
- **Hash pinning.** A daily hash of the deployed code and config, compared with what you shipped. A change you did not make pages you.
- **An emergency address per user**, registered with the watcher, so a compromised platform's balances can be pushed to the users' own wallets. "We sent it to your wallet" beats "it's gone".
- **A kill ladder**, in order: signing switch off, evacuate, remove the key. Never overwrite a key or its decryption secret without backing it up first.

## 8. Web app holes found in a live wallet (Oct 2026 pentest)

A tester with curl and Tor found these in a production hosted wallet in one evening. Each is ordinary web security, and each would have mattered more than any covenant bug.

| Hole | Fix |
|---|---|
| Avatar upload took its type from the `data:` URL header, so `data:image/html` was stored and served as an HTML page on the wallet's own domain (stored XSS, session theft) | Read the type from the file's bytes; serve user files only as png/jpg/gif/webp with a sandbox CSP; 404 on `*.bak*` and server source paths |
| Backup export wrote to a shared `/tmp/<name>_<date>.zip`, so one user could download another user's seed backup | A random path per export, deleted after download |
| Setting a new PIN needed only a session token | Require the password; count wrong PINs toward the lockout |
| "Turn off the send password" needed no re-authentication | Any setting that lowers protection asks for the password or PIN (and, once you have it, the second factor) |
| OAuth cookie had no SameSite and was accepted cross-site on approve pages | SameSite=Lax; accept the cookie on POST only for same-origin requests |
| An invite flow pre-verified the invitee's email, and signup only checked "verified" | Prove the address with a link or code before any claim |
| A payout cron paid accounts that never verified their email | Require verified email before any automatic payout |
| Email codes from `Math.random`, unlimited guesses | `crypto.randomInt`, a guess cap per email, a send cap per hour |
| Non-string JSON fields reached SQL and bcrypt | Type-check every credential field; mysql2 `stringifyObjects` |
| Per-payment `callback_url` not validated (blind SSRF by a registered client) | https only, block private and loopback addresses |

OAuth rules that held up: redirect URIs stored as a list and matched exactly; authorization codes consumed atomically (one UPDATE that succeeds once); refresh tokens rotated on use; the client's allowed scopes enforced on every call; webhooks signed with HMAC, compared in constant time, with a 5-minute timestamp window.

## 9. People and process

- After a compromise: change passwords from a clean device, revoke sessions and API tokens, rotate server keys and JWT secrets, wipe the machine rather than clean it.
- Keep a written plan of what you will press if you wake up to a breach. The goal is a few red buttons, not a night of improvising.
- A second, independent review of a money design (another model or person, given a written packet cold) found the most important bug in one production design, a hot-key path that could create an escrow paying itself (`04-covenant-design-patterns.md`, principle 14). Give the next reviewer the previous triage so it spends its effort elsewhere.

## 10. The Binance case (Oct 2026): what a Kaspa builder should take from it

Binance lost about $6.3 billion from hot wallets on Oct 29, 2026, its own incident report describing a months-long chain. In short, from that report:

1. June: an engineer on the wallet team searched for a popular AI coding tool and downloaded a tampered Homebrew installer. The device was replaced and credentials rotated, but role details and names of colleagues had already leaked.
2. September: a second engineer went through a fake hiring process (video interviews with impersonated HR and executives) and was asked to download an "assignment" repository. Its install-time code replaced part of a legitimate developer tool with malware that endpoint tools did not catch.
3. October: from that machine the attackers reached two more wallet engineers' devices through an internal AI tool. One of those computers held a token with container deployment permissions.
4. With that token they deployed a container, ran a script, and reached the wallet signing service, which then signed withdrawals for them. On Ethereum they also used EIP-7702 delegations, which kept working after withdrawals were paused.

What to take from it, for any Kaspa service that signs:

- **The signer is the target, and developer machines are the way in.** No deploy token, cloud credential or SSH key that can reach the signer should sit on a laptop used for browsing, interviews or trying new tools. Deploys to the signer box go through a separate, narrow path.
- **Install tools only from the vendor's own site or repository**, never from a search result or an ad, and check the checksum where one is published. This applies to AI coding tools too.
- **Never run install-time code from a repository someone sent you** (a "take-home test", a "fix for your bug", a sample project). Read `package.json` scripts, `setup.py`, Makefiles and shell scripts first; use `npm install --ignore-scripts` for code you have not read.
- **An AI tool with access to several machines is a bridge between them.** Give coding agents and internal AI tools the narrowest access that works, never standing access to production or signing infrastructure, and log what they run.
- **A paused service does not revoke powers on chain.** On Kaspa the equivalent of a lingering delegation is a key named in a covenant: it keeps its power until the coin moves. Have an emergency path (evacuate to a rescue wallet) for every coin a hot key can touch, and practise it.
- **Velocity caps and intent binding (section 2) are what limit the damage** when everything before them has failed.
