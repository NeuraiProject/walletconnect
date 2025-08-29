# Neurai × WalletConnect v2 — Integration Guide

This document walks you through everything needed to get **Neurai (XNA)** working with **WalletConnect v2**.

It’s written as a checklist so you can tick things off as you go. The idea is to make it easy to follow whether you’re building a wallet, a dApp, or just experimenting.

---

## 0. Background (read this first)

* WalletConnect v2 works with *namespaces* to negotiate permissions between wallets and dApps.
* Chain IDs follow [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md). For Bitcoin and forks, the namespace is **`bip122`**, and the reference is the first 32 hex chars of the genesis block hash.
* Accounts follow [CAIP-10](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-10.md): `namespace:reference:address`.
* Neurai is a UTXO chain (similar to Bitcoin) with its own genesis block, RPC ports, and BIP32 derivation paths.

---

## 1. Define Neurai in CAIP terms

* [x] Get the **genesis block hash**:
  `00000044d33c0c0ba019be5c0249730424a69cb4c222153322f68c6104484806`
* [x] Build the **Chain ID (CAIP-2)**:
  `bip122:00000044d33c0c0ba019be5c02497304`
* [x] Example of a **CAIP-10 account** for Neurai:
  `bip122:00000044d33c0c0ba019be5c02497304:RXNA...`

---

## 2. RPC methods to expose in the wallet

We need a small JSON-RPC surface so dApps can interact with Neurai through WalletConnect. Maybe blockbook or neurai-rpc based ravenrebels version. Suggested methods:

* [x] `neurai_getAddresses` → return a list of addresses (with CAIP-10)
* [x] `neurai_getUtxos` → return UTXOs for one or more addresses
* [x] `neurai_signMessage` → sign an arbitrary message
* [x] `neurai_signPsbt` → sign a PSBT v2 (base64)
* [x] `neurai_finalizePsbt` → finalize the PSBT (optional)
* [x] `neurai_broadcastTransaction` → push a raw hex transaction to the network

> Tip: stick to **PSBT** as the only way to sign transactions. It keeps things clean and avoids a lot of edge cases.

---

## 3. WalletConnect namespace setup (dApp side)

When your dApp proposes a session, it should request the Neurai namespace like this:

```json
{
  "requiredNamespaces": {
    "bip122": {
      "chains": ["bip122:00000044d33c0c0ba019be5c02497304"],
      "methods": [
        "neurai_getAddresses",
        "neurai_getUtxos",
        "neurai_signMessage",
        "neurai_signPsbt",
        "neurai_broadcastTransaction"
      ],
      "events": ["accountsChanged", "chainChanged"]
    }
  }
}
```

The wallet then accepts and mirrors this in its `sessionNamespace`.

---

## 4. Wallet implementation (what you need on the wallet side)

* [x] Get a `projectId` from the WalletConnect dashboard
* [x] Integrate [WalletKit / Sign SDK](https://docs.walletconnect.com/)
* [x] Listen for **session proposals** and approve them if the chain ID matches Neurai
* [x] Handle `session_request` for each `neurai_*` method
* [x] For PSBTs: decode base64 → sign inputs → return signed PSBT base64
* [x] For broadcasting: send the raw hex transaction to a Neurai RPC node
* [x] Derivation: use BIP32 path for Neurai (official is `m/44'/1900'/0'/0`)

---

## 5. dApp implementation

* [x] Use WalletConnect **Universal Provider / Sign Client**
* [x] Propose a session with the namespace above
* [x] Call `neurai_getAddresses` after connecting, and show the user’s CAIP-10 account
* [x] Build PSBTs locally:

  * Fetch UTXOs with `neurai_getUtxos`
  * Construct a PSBT v2 in your dApp
  * Ask the wallet to sign with `neurai_signPsbt`
  * Finalize and broadcast
* [x] Watch for `accountsChanged` and `chainChanged` to keep your UI in sync

---

## 6. Register Neurai in WalletConnect ecosystem (optional)

* Submit a PR/issue to the WalletConnect chain registry with:

  * CAIP-2 Chain ID
  * Name, logo, explorer, RPC URLs
  * Supported JSON-RPC methods

This helps other wallets and dApps discover Neurai.

---

## 7. Node setup for testing

* Mainnet RPC: `19001`
* Testnet RPC: `19101`
* Regtest RPC: `19201`
* Always verify the genesis block hash matches the Chain ID you’re using.

---

## 8. Security & UX tips

* Always use **PSBT signing** — never let the dApp craft raw signatures directly
* Show users the **transaction details** before signing
* Support multiple accounts (different derivation paths)
* Handle reconnects and session persistence (topics can expire)

---

## 9. Code snippets

### Session proposal (dApp)

```ts
import { SignClient } from "@walletconnect/sign-client";

const client = await SignClient.init({ projectId: "<PROJECT_ID>" });

const requiredNamespaces = {
  bip122: {
    chains: ["bip122:00000044d33c0c0ba019be5c02497304"],
    methods: [
      "neurai_getAddresses",
      "neurai_getUtxos",
      "neurai_signMessage",
      "neurai_signPsbt",
      "neurai_broadcastTransaction"
    ],
    events: ["accountsChanged", "chainChanged"]
  }
};

const { uri, approval } = await client.connect({ requiredNamespaces });
if (uri) {
  // Display QR code or modal here
}

const session = await approval();
// session is ready
```

### Handling requests (wallet)

```ts
signClient.on("session_request", async (event) => {
  const { topic, params, id } = event;
  const { request } = params;
  const { method, params: rpcParams } = request;

  let result;
  switch (method) {
    case "neurai_getAddresses":
      result = await getAddresses();
      break;
    case "neurai_getUtxos":
      result = await getUtxos(rpcParams);
      break;
    case "neurai_signPsbt":
      result = await signPsbt(rpcParams.psbtBase64);
      break;
    case "neurai_broadcastTransaction":
      result = await broadcastTx(rpcParams.hex);
      break;
    default:
      throw new Error("Unsupported method");
  }

  await signClient.respond({
    topic,
    response: { id, jsonrpc: "2.0", result }
  });
});
```

---

## 10. Final checklist before release

* [ ] Wallet side: approve sessions, respond to `neurai_*` methods
* [ ] dApp side: connect, build PSBTs, request signing, broadcast
* [ ] Chain ID documented: `bip122:00000044d33c0c0ba019be5c02497304`
* [ ] Account format documented (CAIP-10)
* [ ] Node running with RPC enabled for testing
* [ ] (Optional) registered in WalletConnect chain registry

---
## 11. Websites related

