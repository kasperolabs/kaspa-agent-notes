# 15. Supporting several wallets

How to let users connect with whichever Kaspa wallet they have: browser extensions (Kasware, Kaspire, Kastle), hosted web wallets (Kasla and similar) and phone wallets (Kasanova, Kurncy and others). Every call below is taken from wallet-connect code running in production on mainnet (Nov 2026) unless marked **practice**. Wallets update often: when one stops behaving as described, check its current docs and say so in an issue here.

Read with `01-wallet-safety.md` (what can go wrong) and `02-sending-and-signing-kas.md` section 5 (the send and sign call shapes).

## 1. The shape: one adapter per wallet

Write one small object per wallet with the same methods, and never branch on wallet names in the rest of the app.

```js
// Every adapter answers the same questions. `can` decides what the UI offers.
// { id, label, detect(), connect() -> address, activeAddress(), onAccountChange(fn),
//   signMessage(msg) -> signature, sendKas(to, sompi) -> txid, signPskt(txJsonString, indexes),
//   can: { send, signMessage, signCovenant } }
```

| Wallet | Kind | Send KAS | Sign a message (login) | Sign a covenant input |
|---|---|---|---|---|
| Kasware | extension | yes | yes | yes (`signPskt`) |
| Kaspire | extension, request-style | yes | yes (returns an object) | yes (`signPskt` with `sender`) |
| Kastle | extension | yes | yes | **no** at the last test (Nov 2026): it returns the transaction with the covenant input unsigned, and no error |
| Hosted web wallet (Kasla) | no extension; login by redirect | yes, through an approval page on the wallet's site | no; login is an OAuth-style redirect | yes, through a signing-request approval page; the wallet never broadcasts |
| Phone wallets (Kasanova, Kurncy) | separate app | the user sends by scanning an address | no page connection | no |

Offer only what a wallet can do on this page. A page that must sign covenant spends hides Kastle (or explains it) instead of letting the user reach the last step and fail.

## 2. Detection

```js
const hasKasware = () => typeof window.kasware !== 'undefined';
const hasKastle  = () => typeof window.kastle !== 'undefined';
const hasKaspire = () => typeof window.kaspire !== 'undefined' && window.kaspire !== null && window.kaspire.isKaspire === true;
const isMobile   = () => /Android|iPhone|iPad|iPod/i.test(navigator.userAgent);
```

- Check when the user clicks Connect, not only at page load: extensions inject their object at their own pace.
- Desktop extensions do not exist in phone browsers. On a phone, offer the hosted wallet and the QR flow (section 7), not extension buttons that cannot work.
- Kaspire is a request-style provider: one `request({ method, params })` instead of named methods.

```js
function kaspireRequest(method, params) {
  const req = { method };
  if (params !== undefined) req.params = params;
  return window.kaspire.request(req);
}
```

## 3. Connecting (getting the address)

Ask silently first, prompt only if needed.

```js
async function connectKasware() {
  let accounts = await window.kasware.getAccounts().catch(() => []);
  if (!accounts.length) accounts = await window.kasware.requestAccounts();   // shows the wallet's prompt
  let address = accounts[0];
  if (address && !address.startsWith('kaspa:')) address = 'kaspa:' + address; // normalize
  return address;
}

async function connectKastle() {
  await window.kastle.connect('mainnet');            // the network is part of the connect call
  const account = await window.kastle.getAccount();
  return account.address;
}

async function connectKaspire() {
  let accounts = await kaspireRequest('getAccounts').catch(() => []);
  if (!accounts || !accounts.length) accounts = await kaspireRequest('requestAccounts');
  const address = accounts && accounts[0];
  if (typeof address !== 'string' || !address.startsWith('kaspa:')) {
    throw new Error('Switch Kaspire to Kaspa Mainnet and try again');   // a testnet address starts kaspatest:
  }
  return address;
}
```

- **Kaspire rejects at once when it is locked**; it does not open an unlock window. Tell the user to unlock it.
- **The network.** Kastle takes it in `connect`. For Kaspire, the address prefix tells mainnet (`kaspa:`) from testnet (`kaspatest:`). Kasware can be switched to the IGRA L2, and the address alone does not show that. The production code does not call a network API for Kasware; it tells the user to check the network name in the wallet before signing (`01-wallet-safety.md` section 3).

## 4. Login: sign a challenge, verify on the server

A connected address proves nothing. Login is a signature over a one-time challenge your server issued.

```js
// 1. server: issue a nonce for this address and this site
const { nonce } = await (await fetch('/api/auth/nonce', { method: 'POST', headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ address, site: location.origin }) })).json();

// 2. the wallet signs it
let signature;
if (walletId === 'kasware') signature = await window.kasware.signMessage(nonce);
if (walletId === 'kastle')  signature = await window.kastle.signMessage(nonce);
if (walletId === 'kaspire') {
  const r = await kaspireRequest('signMessage', { address, message: nonce });
  signature = r && r.signature ? r.signature : r;            // Kaspire returns { address, signature }
}

// 3. server: verify, then issue a session
await fetch('/api/auth/connect', { method: 'POST', headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ address, message: nonce, signature, walletType: walletId, site: location.origin }) });
```

Server side:

- Verify with the SDK's `verifyMessage` against the 32-byte key carried in the `kaspa:q…` address. Accept every signature shape seen: Kasware 0.11.0 returns 128 hex characters, older Kasware 88 characters of base64, Kaspire an object (unwrap `.signature`). Proven on mainnet after a Kasware update broke a login that accepted only one shape.
- **Practice:** a nonce is single-use, expires in minutes, and is bound to the site it was issued for. Put the site's origin in the signed text so a signature from one site cannot log into another.
- Issue a session token (a signed JWT, for example) and verify it on every request that touches money. The address in the verified token is the user; never an address the browser sends later (`02-sending-and-signing-kas.md` section 7).

## 5. Keep the session on the wallet's account

```js
async function activeAddress(walletId) {
  if (walletId === 'kasware') { const a = await window.kasware.getAccounts(); return (a && a[0]) || null; }
  if (walletId === 'kastle')  { const i = await window.kastle.getAccount();   return (i && i.address) || null; }
  if (walletId === 'kaspire') { const a = await kaspireRequest('getAccounts'); return (a && a[0]) || null; }
  return null;
}

function watchAccountSwitch(onSwitch) {
  const h = accounts => onSwitch(Array.isArray(accounts) ? accounts[0] : (accounts && accounts.address) || null);
  try { if (window.kasware && typeof window.kasware.on === 'function') window.kasware.on('accountsChanged', h); } catch (_) {}
  try { if (window.kastle && typeof window.kastle.on === 'function') window.kastle.on('accountsChanged', h); } catch (_) {}
  try { if (hasKaspire() && typeof window.kaspire.on === 'function') window.kaspire.on('accountsChanged', h); } catch (_) {}
}
```

- On an account switch, end the session and reconnect.
- Before every signature, compare `activeAddress()` with the session's address and refuse on a mismatch. A real payout once went to the wrong party because the extension had switched accounts after login (`08-failures-log.md`, Wallets).

## 6. Hosted web wallets (no extension)

A hosted wallet (Kasla is one) holds the user's key on its own servers, so there is nothing in the page to call. Two flows:

**Login by redirect.** Your page sends the user to the wallet's connect URL with a return address; the wallet sends them back with a token.

- **Practice:** before redirecting, store a random `state` value in `sessionStorage` and send it along; on return, refuse a token that does not come back with the same `state`. Otherwise anyone can send a user a link that logs them into the attacker's account.
- Verify the returned token on your server (or with the wallet's verify endpoint) before treating the user as logged in, and remove the token from the address bar at once (`history.replaceState`). Prefer a token in the URL fragment (`#`) over the query string, which ends up in server logs.

**Payments and signatures by approval page.** Your server creates the request with the wallet (payee, amount, memo, or a transaction to sign with the input indexes), gets back an approval URL, and the page opens it.

- Open the popup inside the click handler (`window.open` first, then point it at the URL), or the browser blocks it.
- The user approves on the wallet's own site. Your page learns the result by polling the request's status for the transaction id.
- For a signing request the wallet returns the signed transaction and never broadcasts; your code assembles any covenant signature script and broadcasts it.

## 7. Phone wallets and "any wallet"

A phone wallet cannot be called from a desktop page, and most cannot tell a page which address they control.

- Show the receiving address as a QR code and as text with a copy button, and show the exact amount in KAS beside it. **Practice:** encode the plain address in the QR. No Kaspa payment-link format with an amount was found that wallets agree on, so do not count on an amount inside the QR.
- Learn that the payment arrived by watching the address (`16-accepting-payments.md`). To tell orders apart, give each order its own address, or a slightly different amount.
- If you must route such payments into a per-user covenant, one working pattern is a one-time intake address that your server forwards from within seconds. That makes your server briefly custodial (it holds the intake key): say so to users, and treat it under the third row of the tiers in `00-agent-rules.md`.

## 8. If you would rather not build this

A hosted connect widget already does detection, login and the hosted-wallet flow for Kasware, Kastle, Kaspire and Kasla (KasperoConnect, served by kasperopay.com, with a merchant id from its operator). Using one is a trust decision: the widget's operator issues your users' sessions.

## 9. Test each wallet before you list it

- A fresh browser profile with only that wallet installed.
- The wallet locked; the user rejecting the prompt; the user closing the popup.
- The wrong network (testnet, IGRA).
- Switching accounts after login, then trying to pay.
- A phone, and a popup blocker.
- One real send and, if the page needs it, one real covenant signature on mainnet with a tiny amount.
