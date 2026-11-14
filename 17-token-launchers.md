# 17. Building a token launcher

Practices for any site or tool that lets people create KCC-20 tokens. Learned from running a launcher on mainnet through the KCC-20 launch (Oct 2026) and from indexing every other launcher's tokens. Read with `05-kcc20-tokens.md` (how tokens work on chain) and `06-redteam-before-deploy.md`.

A launcher is where most token scams start: a copied name, a hostile string in metadata, an icon that is really a page. Everything a launcher accepts will be shown by wallets and explorers you do not control. Treat every field as an attack on them.

## 1. Names and tickers

- **Normalize before comparing:** Unicode NFKC, Cyrillic and Greek lookalikes folded to Latin, digit swaps (0/O, 1/I, 5/S), accents dropped, letters and digits only. Compare that "skeleton", never the raw string. `NАCHO` with a Cyrillic А is `NACHO`.
- **Reserved names are refused:** the majors (Bitcoin, Kaspa, Ethereum, wrapped assets), stablecoin names (USD, USDT, USDC and bridged variants), and existing community tokens from earlier Kaspa token standards. Keep the list in a file you can extend without a deploy, and check names, tickers and words inside names. Publishing your list lets other launchers reuse it.
- **First come, first served:** a name already launched is refused for everyone except its original launcher, who may launch numbered siblings (`PRIZE1`, then `PRIZE2`).
- **Refuse names that cannot be told apart from nothing:** whitespace-only or punctuation-only names (a token named `" "` was launched), names with no letter at all, and names that look like an indexer's placeholder for an unnamed token (`Token 1a2b3c4d`, `STD-…`).
- **Refuse without saying which rule matched.** "This name is not available" is enough; a detailed reason teaches a scammer which character to change.
- Strip and refuse control characters, zero-width characters and right-to-left override characters in names and tickers. They let `TOKEN` and `NEKOT` look alike on screen.

## 2. Every other open field

- The safest launcher has no free-text fields beyond name and ticker. One launcher dropped description and website before launch day because they were attack vectors that helped neither wallets nor indexers.
- If you keep them: plain text only, a length cap per field, HTML and markdown stripped (not escaped and stored; stripped), control, zero-width and bidi characters removed, and URLs limited to `https://` with a host. Store what you accepted, not what was sent.
- Remember the reader: wallets and explorers will render these strings with code you did not write. Assume one of them does not escape HTML. What you refuse at the door is the only protection their users get from you.
- On your own pages, show links from metadata as text, never as clickable links.

## 3. Images

- A size cap on upload (a few MB), then **decode and re-encode** to one raster format at a fixed square size, with metadata stripped. Never store the original bytes.
- **Never accept, store or serve SVG.** It can carry script. Read the file type from the bytes, never from the name or a `data:` header (a `data:image/html` upload once became stored XSS on a wallet's own domain).
- Decide on animated images before launch. One launcher found animated icons annoying in wallets and was still undecided; whatever you choose, make it a rule.
- Pin the image by its hash. Put the hash (or the image) in the on-chain metadata at genesis, so anyone can check that the picture a wallet shows is the one the launcher committed to.

## 4. What goes on chain

- **Metadata at genesis** (KCC-23: in the transaction the genesis's authorizing input spends), with the image hash. No links that an indexer must fetch, no off-chain registration step.
- Anything you want to prove about a launch (an endowment, an allocation, a fee paid) goes in the genesis transaction itself, where everyone can read it.
- **A unique value in every per-token covenant** (the token's covenant ID, for example). Two launches with the same terms otherwise produce the same script and the same address: two test tokens once shared one miner-drip covenant.

## 5. The program and its powers

- **Use the pinned reference KCC-20 build** unless you have a reason not to. Wallets and indexers read it; a variant of your own is invisible to them until they pin it (`05-kcc20-tokens.md` section 2).
- **No issuer powers by default.** A launcher should not hand a token's creator the power to freeze, burn, seize or mint on holders' coins. If a token type has such powers by design (a regulated stablecoin), the launch page and the token page say so up front.
- **Red-team every covenant your launcher deploys** (pools, curves, drips, minters, sale contracts). A keyless drip used at launch shipped on mainnet without `require(tx.inputs.length == 1)`; two equal coins could be merged and the second paid to the miner (`08-failures-log.md`).
- Publish your programs' source, or at least their template hashes, so indexers can recognize and label your tokens.

## 6. The launch flow

- Show the full cost before the signature: network fees, any platform fee, any endowment, the KAS each token coin carries.
- Say "launched" only after a node shows the genesis accepted and the covenant ID exists.
- Tell launchers plainly that a wallet shows a new token only once its indexer reads the token's program and metadata. A token can be real and still invisible in a wallet for a while.
- Keep a launch record (genesis txid, covenant ID, metadata, image hash) the launcher can download.

## 7. Spam and junk

- A free launcher gets flooded. A launch fee or a deposit filters out "let's launch 1,000 tokens and see". Weigh the price against honest small launchers.
- Expect test tokens. Name them clearly on your own pages, or ask launchers to use testnet. Indexers filter names like "test" and tokens whose supply sits at a burn address; make that easy for them.

## 8. Checklist

- [ ] Names and tickers checked by skeleton against the reserved list and every launched name; numbered siblings only for the original launcher.
- [ ] Whitespace-only, letterless, placeholder-looking and invisible-character names refused, with no reason shown.
- [ ] Free-text fields dropped, or plain text with caps, stripped of HTML and invisible characters, URLs `https://` only.
- [ ] Images re-encoded to a fixed raster, typed from their bytes, SVG refused, hash pinned on chain.
- [ ] Metadata and image hash in the genesis; per-token covenants carry a unique value.
- [ ] Reference build, no hidden issuer powers, every launch covenant red-teamed.
- [ ] Full cost shown before signing; "launched" only after the node shows the genesis.
