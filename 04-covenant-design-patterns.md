# 04. Designing covenants: principles and patterns

A covenant is a Kaspa address with spending rules attached. Anyone can send KAS to it. Taking KAS out works only when the rules say so: who must sign, after which time, to which address, with which secret.

Use the words "covenant" and "spend paths", not "smart contract". A covenant is a set of rules on coins, not a program with storage and accounts. That framing misled even experienced builders about what the code can and cannot do.

The examples below compile with current SilverScript and pass `kaspa-redteam` with every path explored. The originals, in broken and fixed versions, are in github.com/kasperolabs/kaspa-redteam/tree/main/examples (MIT).

## 1. Start with sentences, not code

Before writing any SilverScript, write down in plain words, for every party and every case:

- who can take the money, when, and to where;
- what happens if a party disappears, loses their key, or refuses to sign;
- what a stranger can do with this coin (the answer should be "nothing" or "only push it along its fixed path");
- how much fee any path may burn.

Then number the sentences. For each sentence write one spend that must be accepted and one that must be refused, and run both on the script engine. If the engine cannot make a sentence true, change the sentence first, then the script. This "invariants first" method caught every issue below before anything reached mainnet in a six-contract wallet project.

## 2. Principles that would have prevented every bug seen so far

1. **Pin the recipient.** Check `tx.outputs[i].scriptPubKey` against a constructor value or a key the rules name. A path that checks only a signature lets the signer, or anyone reusing that transaction shape, send it anywhere. (Attack OUT-01.)
2. **Pin the amount.** Check `tx.outputs[i].value` against the coin's own value, less a bounded fee. Otherwise output 0 can be cut to a sliver. (OUT-02, FEE-01.)
3. **Check every output that matters, not just the first.** (OUT-03.)
4. **Tie payouts to the coin's value.** Fixed payouts with nothing tying them to `tx.inputs[this.activeInputIndex].value` leave the rest to whoever builds the transaction. (OUT-04.)
5. **Read the active input, not input 0.** Use `this.activeInputIndex`. An attacker can put their own coin at input 0. (IN-02.)
6. **One coin per spend, unless the design needs more.** Add `require(tx.inputs.length == 1);` when the contract assumes it is spent alone. Without it, two coins of the same covenant can share one set of outputs and the second coin becomes miner fee. This shipped once on mainnet (MinerDrip v1) and was fixed in v2. (IN-01.)
7. **Keys come from the constructor, never from the caller.** `entry arbitrate(pubkey judge, sig s) { require(checkSig(s, judge)); }` lets anyone be the judge. (SIG-02, ORC-04.)
8. **A secret is not an authorization.** When a secret is revealed in the mempool, anyone can copy it. Require the claimer's signature too, or fix the whole spend (destination and amount) so copying gains nothing. (SEC-01.)
9. **Unsigned paths must fix the whole spend.** A path anyone can trigger (timeout refund, drip, inheritance) must pin destination, amount, input count and output count.
10. **Bound the fee.** `require(out.value >= in.value - maxFee)`. Budget `maxFee` for the redeem script size you ship plus any change output.
11. **A covenant never pulls money from anyone.** "X pays me monthly" is a deposit only X's wallet can make. Model deposits and spends separately.
12. **Permissionless escapes.** Every coin should have a path that eventually returns it to someone (a timeout refund, a recovery key) so money is never stuck forever if a party vanishes.
13. **Add nothing the covenant lacks.** A page or app built on a covenant must not invent rules the covenant does not enforce, and must say plainly what the covenant allows (for example, that an owner path can take everything, even while a payee's request is pending).
14. **Whatever one hot key can create alone gets fixed terms.** If a service key may open an escrow, a hold or a payment coin by itself, the parent covenant must fix that coin's terms: refundable, pinned arbiter, an expiry no earlier than the capture window, and silence resolving to the payer. Otherwise a breached service key writes a hold with itself as payee, silence-to-payee and a short expiry, and the "nobody signs" expiry path pays it. Found by an independent review of a design that had passed thousands of engine tests (Nov 2026): every test was honest-service.
15. **A covenant pins addresses at creation; never retire them while it lives.** A vault whose release pays a checking address pinned at deploy keeps paying that address. Do not rotate, migrate or delete a key or address a live covenant pays to.
16. **Emergency paths get room on the fee.** Rescue and evacuate run against a clock, possibly in a fee spike; give them a higher `MAX_FEE` than routine paths (about 0.5 KAS for emergencies works; keep routine release tight).

## 3. Patterns

### Time-locked vault / dead man's switch

```
contract LastWill(pubkey owner, pubkey heir, int inactiveDaa, int maxFee) {
    entry inherit() {
        require(this.ageDaa >= inactiveDaa);
        require(tx.inputs.length == 1);
        require(tx.outputs.length == 1);
        require(tx.outputs[0].scriptPubKey == byte[](new ScriptPubKeyP2PK(heir)));
        require(tx.outputs[0].value >= tx.inputs[this.activeInputIndex].value - maxFee);
    }
    entry spend(sig s) {
        require(checkSig(s, owner));
    }
}
```

Anyone can trigger `inherit` after the period, but only toward the heir. The owner resets the clock by spending (and redepositing). `inactiveDaa` is in DAA: 30 days on mainnet is 25,920,000.

### Escrow with an arbiter

```
contract Escrow(pubkey buyer, pubkey seller, pubkey arbiter) {
    entry release(sig s) {
        require(checkSig(s, buyer));
        require(tx.outputs[0].scriptPubKey == byte[](new ScriptPubKeyP2PK(seller)));
    }
    entry arbitrate(sig s) {
        require(checkSig(s, arbiter));
        require(
            tx.outputs[0].scriptPubKey == byte[](new ScriptPubKeyP2PK(seller))
                || tx.outputs[0].scriptPubKey == byte[](new ScriptPubKeyP2PK(buyer))
        );
    }
}
```

Nothing on Kaspa arbitrates by itself. An arbiter is just a third key someone must hold and use. If no real arbiter exists, make the arbiter optional and design a two-path contract (release by both, refund after a timeout). A placeholder arbiter key leaves its paths dead.

### Hash-locked claim (HTLC, password gift)

```
contract HtlcSwap(byte[32] hash, pubkey recipient, pubkey sender, int timeoutDaa) {
    entry claim(byte[] secret, sig s) {
        require(sha256(secret) == hash);
        require(checkSig(s, recipient));
    }
    entry refund(sig s) {
        require(this.ageDaa >= timeoutDaa);
        require(checkSig(s, sender));
    }
}
```

For a "first person with the password" gift where the claimer is not known in advance: derive a keypair from the password in the browser (a slow KDF such as Argon2id with a random salt carried in the link), lock the coin to that derived key, and have the claim page sign with it. Never send the password to a server. Add fees on top of the gift so the claimer receives the round amount. With AI around, a riddle that one good question can solve will be solved in seconds.

### Recurring payment / allowance (change back to itself)

```
contract Allowance(pubkey funder, pubkey recipient, int installment, int periodDaa, int maxFee) {
    entry pay() {
        require(this.ageDaa >= periodDaa);
        require(tx.inputs.length == 1);
        require(tx.outputs.length == 2);
        require(tx.outputs[0].scriptPubKey == byte[](new ScriptPubKeyP2PK(recipient)));
        require(tx.outputs[0].value == installment);
        require(tx.outputs[1].scriptPubKey == tx.inputs[this.activeInputIndex].scriptPubKey);
        require(tx.outputs[1].value >= tx.inputs[this.activeInputIndex].value - installment - maxFee);
    }
    entry cancel(sig s) {
        require(checkSig(s, funder));
    }
}
```

`tx.outputs[1].scriptPubKey == tx.inputs[this.activeInputIndex].scriptPubKey` sends the rest back to the same covenant with the same rules. The change coin's relative age restarts at zero, which is what makes the period recur.

### All-or-nothing pledge

```
contract Pledge(pubkey creator, pubkey backer, int deadlineDaa, int maxFee) {
    entry claim(sig s) {
        require(checkSig(s, creator));
    }
    entry refund() {
        require(this.ageDaa >= deadlineDaa);
        require(tx.inputs.length == 1);
        require(tx.outputs.length == 1);
        require(tx.outputs[0].scriptPubKey == byte[](new ScriptPubKeyP2PK(backer)));
        require(tx.outputs[0].value >= tx.inputs[this.activeInputIndex].value - maxFee);
    }
}
```

A full design adds a goal check on `claim` (the claim transaction's output to the creator must be at least the goal, across many pledge coins spent together). One covenant per backer keeps refunds simple. A platform fee in a pledge contract is mechanically the same as a backdoor for whoever holds the fee key: disclose it.

### Revenue split

```
contract RevenueSplit(pubkey artist, pubkey label, int maxFee) {
    entry split() {
        require(tx.inputs.length == 1);
        require(tx.outputs.length == 2);
        int total = tx.inputs[this.activeInputIndex].value - maxFee;
        int artistShare = total * 7 / 10;
        require(tx.outputs[0].scriptPubKey == byte[](new ScriptPubKeyP2PK(artist)));
        require(tx.outputs[0].value >= artistShare);
        require(tx.outputs[1].scriptPubKey == byte[](new ScriptPubKeyP2PK(label)));
        require(tx.outputs[1].value >= total - artistShare);
    }
}
```

Watch integer overflow on `value * bps`. Split the multiplication: `whole = v / 10000; rest = v % 10000; fee = whole * bps + (rest * bps) / 10000;`.

### Per-payment escrow (merchant payments)

One coin per payment at an address fixed by its arguments. A unique `order_hash` constructor argument gives every payment its own address (keep it in the script with a check that always holds, for example that the spent outpoint's txid is not equal to it). Paths: release to the merchant after a window (anyone may run it, so a keeper can), refund to the payer (merchant signs, partial refunds continue to the same script), early release (merchant and payer together). A platform fee below about 0.2 KAS should be skipped rather than paid as a tiny output, because of storage mass.

Warning written into that template, worth copying: never let a token coin be owned by this script's hash. No path checks token outputs, so a path anyone can run would let anyone move such tokens.

### Keyless miner drip (endowment)

A coin with no key that pays a small prize to whoever claims it on a schedule (for example 1/1000 of the balance per period, halving as it drains). Pin: the claim output back to itself, the prize, the period (relative lock), one input per spend. Version 1 of such a contract on mainnet lacked `tx.inputs.length == 1`, so two equal top-ups could be merged into one output and the second paid entirely to the miner. Stamp the token's covenant ID (or another unique value) into each drip so two drips with the same terms do not share one address.

### Vault with an airlock ("not you?")

Withdrawals from a savings vault first land in an intermediate covenant for a fixed window (24 hours, a relative lock). Two paths on the airlock: the owner's key after the window, and an evacuation path to a pre-registered backup wallet at any time. A watcher emails "not you? click here" on every release; clicking evacuates. Proven on mainnet. Design notes:

- Anyone can send coins (or token cells) to a vault address. Handle stray coins one by one; never let a stray coin break the vault's own spends.
- If the service that runs the watcher holds keys in the vault, a breach of that service defeats the airlock. Include a key only the user holds.
- **Setup is the trust moment.** Every value (hot key, guardian, rescue address, delay, the airlock's hash) is fixed when the vault is compiled. A breached site at setup can compile with the attacker's rescue key or a 100-DAA delay and still show the right values on screen. Before funding a large vault, check from a machine the site does not control: the rescue address character by character, the delay in both manifests, the guardian key, that the vault's pinned airlock hash is the airlock's real script hash, and that the addresses match where the deposits landed. Keep the `.ksm` files offline.
- **The rescue address comes from the user's own wallet, proven by a signature** (the extension signs a note). Never a typed address, never an address of the hosted wallet itself (a hacked site would send the rescue to itself), and not "send 0.1 KAS from your wallet" either: a user who sends that proof from an exchange binds the exchange's hot wallet as their rescue, and an evacuation then pays an address that will not credit them. If you read the proof's sender from a public API, that API chooses the rescue key.
- Coins below the minimum deposit sent to a vault address become rows that can never be moved economically. Mark them as dust and keep them out of the balance and the withdraw list.
- The watcher is the part that fails quietly: trust, blindness, retries and clocks are in `11-hot-keys-signers-watchers.md` section 6. Proven on mainnet: rescue drills on a 24-hour vault, alert to rescue in one poll.

### Rolling limits (a card-style daily limit)

A covenant that lets a service key charge up to a limit per day, like a debit card, needs the singleton and the CLTV-tied lockTime from `03-silverscript-v1-gotchas.md` section 8. What that leaves (engine-tested, Nov 2026): lockTime can still be backdated, so an idle account can be charged one full limit per idle day in a single sitting (three back-to-back full-limit charges were accepted). Say so in the terms and keep the limit modest. Also requiring lockTime at or above the input's own DAA score is an untested idea.

### Shared budget (many departments)

When several parties draw from one budget, give each its own coin (its own covenant). One shared coin makes them race: the first spend wins and every other pending spend must be rebuilt.

## 4. Covenant IDs and lineage (KIP-20)

- A covenant ID is an optional binding on a transaction output (transaction version 1). It is born in a genesis transaction: `id = BLAKE2b-256 keyed "CovenantID" over the authorizing input's outpoint and every bound output (index, value, script public key)`. It continues when a spend of a bound coin authorizes new bound outputs.
- Consensus blocks forged entry into a lineage, not exit. A spend keeps the binding only if the script checks it (opcodes 0xcb to 0xd6: `OpAuthOutputCount`, `OpAuthOutputIdx`, `OpCovInputCount`, `OpCovOutputCount`, `OpInputCovenantId`, and others). If your covenant should stay one lineage, check the binding on its own outputs.
- An existing coin can never gain an ID. Mint the ID when the covenant is first funded.
- Bound coins cost 32 extra bytes of storage.
- For anything stateful (its address changes every step), the covenant ID is the only stable identity. Wallets, indexers and your own app should track it by ID.
- Example of a lineage-checked transfer:

```
entry transfer(byte[32] new_owner, sig s) {
    require(checkSig(s, pubkey(owner)));
    byte[32] id = OpInputCovenantId(this.activeInputIndex);
    require(OpCovInputCount(id) == 1);
    require(OpAuthOutputCount(this.activeInputIndex) == 1);
    require(OpCovOutputCount(id) == 1);
    int out = OpAuthOutputIdx(this.activeInputIndex, 0);
    require(tx.outputs[out].value >= tx.inputs[this.activeInputIndex].value);
    validateOutputState(out, State { owner: new_owner, name: name });
}
```

## 5. Covenants that hold tokens

- A KCC-20 cell owned by a covenant ID (owner scheme 0x04) moves whenever that covenant is spent in the same transaction. The token program does not ask where it goes. So every entry of the owning covenant that someone could sign must either refuse to share a transaction with the token (`require(OpCovInputCount(tokenCovId) == 0)`) or check the token's outputs itself. Engine-tested: a hot-key entry that ignored the token let the cell go to an attacker.
- A cell owned by the P2SH hash of a stateful covenant (owner scheme 0x03) is stranded after the covenant's first state change, because the address changed. Own cells by covenant ID instead.
- A token coin held by a covenant with borrowing turned on can be bricked or moved by a stranger (attacks BOR-01, BOR-02). Hold borrow-off cells.

## 6. UX rules for covenant pages

- One page per covenant, built as a state machine per viewer: "there is money here for you", "waiting for the other party", "locked until <date>", "nothing to do". One primary action, carrying the amount.
- Show the rules in plain words first; show the source and parameters folded below for those who want them.
- Show units: KAS with decimals, dates in the user's time zone with UTC noted. Never show sompi.
- When a spend is refused by the rules, say so before the user signs.
- Give every party a portable file (the `.ksm` spend map, `09-verify-and-read-the-chain.md`) so they can withdraw without your site.
