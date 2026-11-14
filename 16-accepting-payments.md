# 16. Accepting a KAS payment

The most common thing people build: a shop, a tip jar, a paywall, a ticket. This is the second row of the tiers in `00-agent-rules.md`: the payer's wallet signs, your server holds no key, and public nodes or the REST API are enough. Lines marked **practice** are recommendations, not chain facts.

## 1. The rule everything else follows

**An order is paid when a node shows the coin at your address, never when the browser says so.** A page can report any txid it likes. Check the chain yourself: the right address, at least the right amount, accepted.

## 2. Your receiving address

- Receive to an address whose key your server does not hold (a wallet the merchant controls). Then a breach of the web server cannot spend the takings.
- **Telling orders apart.** Pick one:
  - **One address per order.** Cleanest. Derive receive addresses from the merchant wallet's extended public key on the server, so no spending key is involved (check your SDK version for the call), or let the merchant's wallet hand out fresh receive addresses.
  - **One address, a unique amount per order** (add a few sompi: 25.00000137 KAS). Simple for a small shop. Expire the amount with the order, and match on the exact value.
  - **A covenant per order** (an escrow, `04-covenant-design-patterns.md`) when the money should wait for delivery or allow a refund.
- Do not reuse one address with the same amount for everyone; you cannot tell who paid.

## 3. Taking the payment

**A. Extension wallet on the page.** The wallet sends; your server checks.

```js
// browser
const txid = await window.kasware.sendKaspa(order.address, Number(order.amountSompi), {});
await fetch('/api/orders/' + order.id + '/paid', { method: 'POST', headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ txid }) });     // a hint for the server, not proof
```

Other wallets' send calls: `02-sending-and-signing-kas.md` section 5 and `15-multi-wallet-support.md`.

**B. Any wallet, by QR.** Show the address (as a QR and as text) and the exact amount. Watch the address until the coin arrives or the order expires (section 4).

**C. Hosted web wallet.** Create the payment request with the wallet's API, open its approval page inside the click, and poll for the txid (`15-multi-wallet-support.md` section 6). Then check it like any other payment.

## 4. Checking it on the server

With a node (yours, or a public one through the `Resolver`; `14-sdk-cookbook.md` sections 3 and 4):

```js
async function isPaid(rpc, order, txidHint) {
  const { entries } = await rpc.getUtxosByAddresses({ addresses: [order.address] });
  const coins = entries.map(coinOf).filter(c => !c.covenantId);           // plain KAS only, not token cells
  const match = txidHint
    ? coins.find(c => c.txid === String(txidHint).toLowerCase())
    : coins.find(c => c.amount === BigInt(order.amountSompi));             // unique-amount orders
  if (!match) return { paid: false };
  if (match.amount < BigInt(order.amountSompi)) return { paid: false, short: match.amount };
  return { paid: true, txid: match.txid, amount: match.amount, daa: match.daa };
}
```

- This works while the coin is unspent at the address, which is the case for a receiving address the merchant does not sweep every second.
- Or use the REST API: `GET https://api.kaspa.org/transactions/<txid>` and check that one output pays your address the amount and that the transaction is accepted. Field names are in api.kaspa.org/docs; times there are already milliseconds.
- Follow many orders with `subscribeUtxosChanged` on their addresses instead of polling (`14-sdk-cookbook.md` section 9). For a few orders, polling every few seconds is fine.

**How long to wait (practice).** On mainnet a transaction is accepted within a second or two. For small amounts, accepted is enough. For larger ones, wait a little longer before releasing goods: a reorg deep enough to undo an accepted payment is rare, and the cost of waiting a few more seconds is low. One indexer in production waits 100 confirmations, about 10 seconds at 10 blocks per second.

## 5. Order rules (practice)

- **Quote in KAS, lock it.** If prices are in dollars, convert once at checkout, show the KAS amount, and give the quote a lifetime (10 to 15 minutes). Name the price source (`12-nodes-and-servers.md` section 1).
- **Underpaid:** do not mark paid; show what is missing; let the payer top up to the same address or amount before expiry.
- **Overpaid:** mark paid and record the excess for a manual refund.
- **Late:** a payment after expiry still arrived. Record it and handle it by hand; never lose it silently.
- **One payment, one order.** Store the txid and refuse to attach it to a second order.
- **The payer pays the network fee.** Ask for the exact amount to arrive.
- **Exchanges.** Some refuse to withdraw to a covenant (`kaspa:p…`) address, and their payouts arrive from addresses that are not the customer's. Do not refund to the sender address of an exchange payment; ask for a refund address.
- **Webhooks** to the merchant's system are signed (HMAC, constant-time compare, short timestamp window).

## 6. Tokens (KCC-20)

Token payments do not show up on the receiving address: a token coin is a covenant coin with its own address. Learn them from a KCC-20 indexer or by following the chain (`05-kcc20-tokens.md` section 12), and identify the token by its covenant ID, never by its name.

## 7. What to tell the merchant

- "Your shop page never holds your wallet's key. Money goes straight to your wallet."
- "An order counts as paid when the network shows the payment, not when the customer's browser says so."
- "Refunds are manual: you send them from your wallet."
