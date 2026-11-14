# 01. Wallet safety

How users lose money on Kaspa, and what a wallet, a dApp page or an AI agent must do so they don't. Most real losses on every chain come from signing something the user did not read, not from broken contracts. Kaspa is no different.

Status labels used below: **proven on mainnet**, **proven on testnet**, **engine-tested** (rusty-kaspa script engine, nothing broadcast), **read in source**.

## 1. Keys and seeds

- A seed phrase or private key is the money. Whoever has it can spend everything, now and later.
- Never ask a user for it. Never build a page that collects it. A "claim", "verify your wallet", "sync" or "support" page that asks for a seed is always theft.
- Server-side signers (bots, keepers, hot wallets) keep the key in a file outside the web root and outside git, mode 600, readable only by the process user. Route every signer through one monitored place with a "halt all" switch, so one command stops every hot key.
- When a key must be derived from a seed for a one-off script, write it to a file with mode 600 and shred the file after use. Never print it.
- Test wallets are separate wallets. Never test with the user's main wallet.
- **A locked wallet does not protect a seed stored somewhere else.** Infostealer malware reads seed phrases from text files, screenshots and notes, then restores the wallet on the attacker's machine; the extension's lock screen is irrelevant. Seeds never go in files, screenshots, notes or password managers on a machine that browses the web. Builders are targeted on purpose (fake job interviews, "we'd like to feature your project" messages, test assignments to download), because their machines also hold server keys and deploy tokens.

## 2. Sign only what you read

A signature approves a whole transaction: every input it spends and every output it creates. A drainer does not need a bug in any contract. It needs the user to sign.

### The drainer patterns to expect

| Pattern | What happens |
|---|---|
| Extra inputs | A "claim 100 TOKEN" transaction also spends the user's KAS coins, with an output to the attacker. |
| Change swap | The change output pays an address that is not the user's, while the amounts look right. |
| Partial signature modes | The page asks for SIGHASH_SINGLE, NONE or ANYONECANPAY. The signed input can then be attached to a different transaction. |
| Approve once, drain later | An OAuth or "connect" permission lets the site request signatures without asking each time. |
| Token cells riding along | A token coin the user owns is added as an input. Some token programs check no signature of their own (see section 4). |

### What a wallet or signing page must do

- Show every input and output in plain words: "This spends 12.4 KAS and 50 NACHO from you, and sends 12 KAS to kaspa:qz…(unknown address)."
- Refuse to sign anything but SIGHASH_ALL for requests from outside.
- Keep app requests to a fixed shape (asset, amount, recipient), approved each time, on the wallet's own page. Avoid open allowances ("spend up to X without asking"). If an app needs recurring charges, make it a capped, revocable mandate the user creates on the wallet's own page.
- Flag outputs to addresses the user has never used.
- Never sign a transaction the wallet did not build or fully decode.

### Proven case: a SINGLE|ANYONECANPAY signature moved a user's tokens (engine-tested, Nov 2026)

KRON's older KCC-20 cell layout has an owner type (type 3) that checks no signature of its own. It only requires that some input in the same transaction spends a plain coin locked to the holder's own key. Any valid signature by the holder on any plain coin in that transaction authorizes the token move.

- With SIGHASH_ALL, that signature covers every input and output, so it is a real approval.
- With SIGHASH_SINGLE|ANYONECANPAY (marketplace listings, some PSKT offers), someone else can add the holder's token cells as inputs and send the tokens anywhere. On the engine, a signature the holder made for an unrelated listing moved all 1,000 of the holder's test tokens to an outsider.

Rule for any wallet: SIGHASH_ALL only, and when a transaction spends the user's own coin and also contains a token input the user owns, the approval page must name the tokens leaving.

## 3. Wallet traps that cost real time

- **Check the network before signing.** Kasware can be switched to the IGRA L2 network (the popup header says "IGRA Mainnet"). Signatures made there are useless on Kaspa mainnet. Confirm the network every time.
- **Keep the session and the wallet on the same account.** If the user switches accounts in the extension after logging in, a payout addressed to "the session user" goes to the old account while the new account signs. This happened (a receiver's withdrawal paid the sender). Fix: listen for `accountsChanged` and drop the session; before every signature, compare the extension's active address with the session address and refuse on a mismatch.
- **Kasware shows "Balance Change: 0"** when signing a covenant input, because the input is not the wallet's own coin. That is not an error. Tell the user in advance.
- **Kaspire rejects when locked** instead of popping an unlock window. Tell the user to unlock it.
- **Popups must open inside the click handler**, or the browser blocks them (hosted-wallet approval windows).
- **Wallets that silently do nothing.** As of the last test (Nov 2026), Kastle's dApp signer left covenant (P2SH) input signatures empty with no error. Always check that every input you asked to be signed comes back with a non-empty signature script before broadcasting.

## 4. Tokens and the KAS balance

- A KCC-20 token coin ("cell") is a covenant coin that carries some KAS (often 0.5 to 1 KAS). It must never be counted as spendable KAS or selected as a fee coin. Spending it with a plain KAS builder destroys or strands the token.
- Low-liquidity tokens should not be added into a user's net worth. Their price can move hard on one small trade.
- Tokens can arrive unasked (airdrops, dust, lookalikes). Put them in a "received, not requested" place. Show links from token metadata as text, never as clickable links.
- **Token icons leak IP addresses.** Loading an icon from a URL the token creator chose tells the creator's server who holds the token. Fetch icons server-side, pin them by hash, re-encode them, and serve your own copy. Never render SVG from an untrusted source.
- Sanitize every metadata string (name, ticker, description). Treat it as hostile HTML.
- A name proves nothing. Two tokens can share name, ticker and image and differ only by covenant ID. Show the covenant ID (first 6 and last 6 hex characters at least) wherever a token is shown.
- Some token programs give an issuer power over holders' coins (freeze, burn, seize, mint). That is a property of the program, not of the name. Show it.

## 5. Addresses

- `kaspa:q…` is a plain key address (P2PK). `kaspa:p…` is a script address (P2SH): a covenant. `kaspatest:` is testnet. Never let a user send mainnet KAS to a `kaspatest:` address or the reverse.
- A P2SH address is the hash of the full script with its keys and values. Same script, same address. If a user deploys the same rules with the same keys twice, both deposits land in the same covenant. Renaming the contract does not change the address.
- The keys inside an unspent covenant are invisible on chain. They appear only when the coin is spent. You cannot discover "covenants that name my key" by scanning the chain.
- Address reuse: some wallet conventions (KIP-22) avoid reusing addresses to keep public keys hidden at rest. A user may choose to keep one permanent public address anyway. Respect the choice, explain the trade-off once.
- **An HD wallet is many addresses.** Change goes to new addresses, so one address's balance from an explorer API is not the wallet's balance. Sum the coins of every derived receive and change address. When every output of a transaction goes to the user's own addresses (including their own covenant, vault and invite addresses), it is an internal transfer, not a send: detect that explicitly or history shows phantom payments.
- **Exchanges may refuse to withdraw to a `kaspa:p…` address.** Test a small withdrawal before telling a user to fund a covenant straight from an exchange.
- **A withdrawal from a covenant is per coin.** If a vault releases one whole coin at a time, fund it in the chunks you will want to take out later.
- Verify an address you build yourself against the wallet's own derivation when possible. Kaspa's bech32 variant needs Kaspa-specific prefix expansion (each prefix byte masked `& 0x1f`, then a zero separator) and BigInt arithmetic; Bitcoin's version gives wrong checksums. Use the SDK's functions instead of writing your own.

## 6. Hosted wallets and app permissions

The operator's side of a hosted wallet (signer gate, watchers, money switches, the web app holes a pentest found) is in `11-hot-keys-signers-watchers.md`. This section is what the user should be told and offered.

- A hosted wallet (the server holds the key) is a different trust model. Say so to the user. If the host is breached, any key it holds is exposed.
- Per-transaction approval on the wallet's own origin is the minimum. "Approval or bust."
- A bearer token that can spend is a spend credential. Treat it like a key: short-lived, verified server-side, never in a URL, never in logs.
- Give users a way out that does not depend on your service: a seed they can export, recovery instructions that work if your site is gone.
- **Message signing is a spend credential for logins.** A wallet that signs any message without asking lets a site obtain the user's signature on another site's login challenge. Ask every time, and show the message.
- **Keep scams structurally out of a wallet with accounts.** Recipient search by email or handle should be exact match only (no partial matches, no "did you mean"), so nobody can browse users. Do not let members send each other money requests; requests should come only from parties the wallet can name (registered apps, covenant programs). Invoices between people belong in a merchant tool where the requester is identified.
- A rescue or backup address that a covenant pins must be proven by a signature from that wallet, not typed. A typed address can be one character off, a hacked site can substitute its own, and proving it with a small payment can bind an exchange's hot wallet instead of the user's (see `04-covenant-design-patterns.md`, airlock).

## 7. What to recommend to a user, in plain words

- "Use a separate wallet for testing and for new sites."
- "Read the popup. If it spends more than you expect or sends to an address you don't know, reject it."
- "No real site will ever ask for your seed phrase."
- "Tokens that arrive by themselves are not gifts. Don't click their links. Don't try to sell them on a site you don't know."
- "Check the network name in your wallet before signing."
- "If two tokens have the same name, the covenant ID is the only thing that tells them apart."
