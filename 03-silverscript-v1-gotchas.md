# 03. SilverScript v1: syntax and gotchas

SilverScript is Kaspa's covenant language, compiled by `silverc` (github.com/kaspanet/silverscript). This file covers silverc v1.0.0 (commit 3ed9733) as used in production since Sept 2026. The language keeps changing. Always check `docs/TUTORIAL.md`, `docs/DECL.md` and `docs/CONSTRUCTOR_ARGS.md` **at the commit you compile with**, and pin that commit.

Rule for samples: a `.sil` file found online or in a repo counts only if it says which compiler it compiled on (ideally also that it was spent on mainnet). Code written for the testnet-era compiler (`entrypoint function`, `this.age`, `int` passed to a `tx.time` lock) does not compile on v1.

## 1. The shape of a contract

```
pragma silverscript ^0.1.0;

contract Vault(pubkey owner, int unlockDaa) {
    entry withdraw(sig s) {
        require(checkSig(s, owner));
        require(this.ageDaa >= unlockDaa);
        require(tx.outputs[0].scriptPubKey == byte[](new ScriptPubKeyP2PK(owner)));
    }
}
```

- `entry name(...)` declares a spend path. (`entrypoint function` was the old syntax.)
- One contract per file. Two contracts in one file did not compile together on v1; deploy families as separate files.
- Constructor parameters are fixed at deploy and become part of the script, so they change the address.
- Contract-level `int constant X = ...;` for fixed protocol values (fees, caps). Anything that varies per deployment (keys, amounts, durations) should be a constructor parameter, not an edited constant.
- A bare `if` without `else` is accepted.

## 2. Types are strict

- Hash a pubkey with a cast: `blake2b(byte[](pk))`. Same for `sha256(byte[](secret))`.
- Compare scripts as bytes: `tx.outputs[0].scriptPubKey == byte[](new ScriptPubKeyP2PK(k))`, or `byte[](new ScriptPubKeyP2SH(h))`.
- `.split(n)` returns a tuple: `(byte[] a, byte[] b) = x.split(n);`
- There is no `checkMultiSig`. Write each `checkSig` against a pinned key, or count signatures yourself.
- `checkMsgSig(datasig, byte[32] digest, pubkey)` verifies a BIP340 Schnorr signature over a raw 32-byte digest with a 32-byte x-only key. The digest is an unkeyed BLAKE2b-256 (not a truncated BLAKE2b-512). Proven on mainnet.
- `OpDiv` and `OpMul` are enabled under Toccata (checked, truncating division).

## 3. Time locks: three domains, never mixed

| Form | Unit | Compiles to |
|---|---|---|
| `require(this.ageDaa >= N)` | relative DAA count (`int`), 0 ≤ N < 2^32 | `<N> OP_DUP OP_0 <2^32> OP_WITHIN OP_VERIFY OP_CHECKSEQUENCEVERIFY` |
| `require(tx.daa >= N)` | absolute DAA score (`int`), below 500,000,000,000 | CLTV in the DAA domain |
| `require(tx.time >= t)` | absolute Unix milliseconds (`temporal`), at or above 500,000,000,000 | `<N> OP_DUP <500000000000> OP_GREATERTHANOREQUAL OP_VERIFY OP_CHECKLOCKTIMEVERIFY` (two `OP_DUP`s in front have also been seen) |

The relative and absolute shapes above were checked by compiling with silverc v1.0.0 (3ed9733): relative bytes `<N> 76 00 05 0000000001 a5 69 b1`, absolute bytes `<N> 76 05 0088526a74 a2 69 b0`.

- No time units on `this.ageDaa`. 30 days is `30 * 864000` on mainnet (about 10 DAA per second), not `30 days`.
- `date("2026-10-01T17:00:00")` compiles as **UTC**. A form that asks for "local time" and passes it straight in is off by the user's offset.
- `LOCK_TIME_THRESHOLD` is 500,000,000,000 ms (Nov 5, 1985). Below it is the DAA domain, above it the timestamp domain.
- The spender must set the input's `sequence` (relative lock) or the transaction's `lockTime` (absolute lock) to satisfy the lock, before signing. See `02-sending-and-signing-kas.md`.

### Reading a lock from bytes (if you write a lock reader)

- **Kaspa's opcode table is not Bitcoin's.** Kaspa: `OP_CHECKLOCKTIMEVERIFY = 0xb0`, `OP_CHECKSEQUENCEVERIFY = 0xb1`. Bitcoin: 0xb0 is NOP1, 0xb1 is CLTV, 0xb2 is CSV.
- A "push immediately followed by 0xb0/0xb1" scanner misses v1 locks: v1 puts range checks between the push and the opcode (shapes in the table above). Follow the value through repeated `OP_DUP`, a comparison opcode (0x9f to 0xa5) and `OP_VERIFY`. Anything else resets the scan, and an unrecognized layout should fail closed.
- In a multi-path contract, collect locks only inside the chosen entry's branch. v1 branches start with `OP_DUP <4-byte tag> OP_EQUAL OP_IF` (`76 04 <tag> 87 63`). Pre-v1 multi-entry contracts use `OP_DUP <n> OP_NUMEQUAL OP_IF` (`76 <00|5n> 9c 63`).
- **Misleading node error.** A relative-lock failure reads "locktime requirement not satisfied -- locktime is greater than the transaction locktime: N > 0". The 0 there is the input's `sequence`, not the transaction's lockTime.

## 4. Dispatch tags and the signature script

- v1 compiles a 4-byte dispatch tag into every contract. The redeem script opens `OP_DUP <tag> OP_EQUAL OP_IF` for each entry.
- The tag is derived from the entry's signature (name and parameter types). It is the same across contracts for the same signature. Renaming the contract or a parameter name does not change it; renaming the entry or changing parameter types does.
- Signature script: `<args in declaration order> <tag> <redeem script>`.
- Read tags from the compiler's artifact JSON (`contracts{name}.entries{name}.dispatch_tag`), never compute them by hand.

## 5. Script public keys as a script sees them

A `scriptPubKey` read inside a script (`tx.outputs[i].scriptPubKey`, `tx.inputs[i].scriptPubKey`) carries a 2-byte version prefix:

- P2PK: `0000 20 <32-byte x-only key> ac`
- P2SH: `0000 aa20 <32-byte BLAKE2b-256 of redeem script> 87`

So a `byte[]` payee constructor parameter must include the `0000` prefix. Proven on mainnet.

The P2SH address of a script is `aa20 BLAKE2b-256(redeem script) 87`, bech32-encoded with version byte 8 (`kaspa:p…`). Fully deterministic: same script and arguments, same address.

## 6. Mempool standardness: at most 15 signature operations per P2SH input

Found on testnet in Nov 2026, invisible on the script engine: **the mempool of every rusty-kaspa node refuses a P2SH input whose redeem script contains more than 15 signature opcodes** (counted statically, without running the script). Consensus would accept it, so the engine and red-team tools pass it; no public node relays it.

- `silverc` inlines every helper function, so each `checkSig` call site in the source becomes one opcode in the script. A helper with three `checkSig` branches called from two entries is six.
- Fix pattern: select the key first, check once.

```
pubkey k = hot;
if (who == 1) { k = user; } else if (who == 2) { k = guardian; } else { require(who == 0); }
require(checkSig(s, k));
```

- Merge entries that take the same signers into one entry with a selector byte.
- Count your sig ops before deploy. A contract with 21 sig ops was frozen and test-ready before testnet refused it.

## 7. Constructor arguments

v1 takes constructor arguments as JSON `[{ "kind": ..., "value": ... }, ...]`, in declaration order. Kinds: `int`, `bool`, `byte`, `bytes`, `text`, `array`, `object`.

| SilverScript type | kind | value |
|---|---|---|
| `int` | int | number (for `temporal`, Unix ms) |
| `pubkey` | bytes | 32 bytes |
| `sig` | bytes | 65 bytes |
| `datasig` | bytes | 64 bytes |
| `string` | text | string |
| `byte[]`, `byte[N]` | bytes | array of byte numbers or hex |
| `byte` | byte | number |

`silverc contract.sil --constructor-args ctor.json` (alias `--ctor`). The output is a SilAbiArtifact JSON: `contracts{name}.entries{name}.{dispatch_tag, params}` and `contracts{name}.compiled.{bytecode, template_hash, state_span}`. Constructor parameter types are read from `--ast-only`.

## 8. Stateful contracts (state in the script)

- A contract with state fields keeps its state inside its redeem script. **Its P2SH address changes on every state change.** There is no stable receive address for it. Its stable identity is its KIP-20 covenant ID (see `05-kcc20-tokens.md`).
- `validateOutputState(i, State { ... })` checks that output `i` continues the same template with the new state. `validateOutputStateWithTemplate(...)` checks an output of another, pinned template (prefix, suffix and template hash). `readInputStateWithTemplate(...)` reads another input's state and checks its template hash and P2SH commitment. Use the template variants whenever more than one program can share a covenant domain.
- A limit that must hold across spends (a daily limit) holds only if the contract keeps itself a single coin bound to its covenant ID. Without that, two coins at the same address each spend the full limit (engine-tested).
- Reading `tx.lockTime` inside a script without a CLTV on the same value is exploitable: a spender who finalizes every input (sequence MAX) makes the node ignore lockTime, and the script then reads whatever lockTime they choose. Even with CLTV, lockTime can be backdated. Design windows with that in mind (engine-tested).

## 9. Size, cost and the compiler

- Every spend ships the whole redeem script. Size is a recurring cost for every user on every spend. A 13.6 KB script made every spend of one contract about 0.0036 KAS dearer than its 10 KB predecessor.
- Folding many similar entries into one entry with a path byte cut one 9.4 KB contract to 3.2 KB (66%) without changing behaviour.
- Folding entries changes the dispatch tags. A watcher or indexer that classifies spends by tag (capture versus refund, say) must then read the path byte from the arguments instead, and the template hash changes. Update every reader in the same release. The saving is large only where each copied branch carries heavy checks (token reads and validations); folding a contract whose branches are one output check saved 15%.
- Keep every historical silverc build pinned by hash. A deployed covenant keeps the bytecode its compiler made. To verify or attack a live covenant, use its real bytes, not today's compile.
- The AI prompt or snippet library you use to generate SilverScript must match the installed compiler. A generator "corrected" to newer docs produces code the deployed compiler rejects, and the reverse.
