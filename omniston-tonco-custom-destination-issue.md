# Omniston TONCO resolver custom destination issue 

## Title 
Omniston `buildTransfer` fails with Internal server error when using TONCO resolver and custom destination/gasExcess address.

---

## Summary

When using Omniston SDK **with the TONCO resolver** (this resolver is currently mostly chosen by Omniston for TON → tsTON swap), `buildTransfer` fails with an internal server error **only** if I set `destinationAddress` and `gasExcessAddress` to a **custom address** instead of the user wallet.

- With the same quote, using the **user wallet address** for `destinationAddress` and `gasExcessAddress` works fine.
- With other resolvers / swap routes, using a **custom address** for destination/excess also works.
- According to the SDK docs and types, `destinationAddress` and `gasExcessAddress` are generic `Address` fields and appear intended to be dynamically configurable, so this looks like a bug specific to the **TONCO resolver path**.

---

## Environment

- SDK: `@ston-fi/omniston-sdk` and `@ston-fi/omniston-sdk-react`  
  - Version: `@ston-fi/omniston-sdk@0.7.6` (and corresponding `omniston-sdk-react` version)
- App: Next.js (based on the official `examples/next-js-app`, with a vault-like flow on top)
- Network: TON mainnet

---

## Minimal reproduction

1. Obtain a quote for TON → tsTON where the **TONCO resolver** is selected (this is index 0 in our multi-asset flow, and in practice Omniston tends to choose TONCO for TON → tsTON swaps).
2. Call `omniston.buildTransfer` with something like:

```ts
const tx = await omniston.buildTransfer({
  quote, // TON -> tsTON via TONCO resolver
  sourceAddress: {
    address: "<USER_WALLET_ADDRESS>",
    blockchain: Blockchain.TON,
  },
  destinationAddress: {
    // custom contract address
    address: "<CUSTOM_CONTRACT_ADDRESS>",
    blockchain: Blockchain.TON,
  },
  gasExcessAddress: {
    // same custom contract address
    address: "<CUSTOM_CONTRACT_ADDRESS>",
    blockchain: Blockchain.TON,
  },
  // refundAddress omitted (same as docs: optional)
  useRecommendedSlippage: false, // or the recommended flag from quote
});
```

3. The call to `buildTransfer` rejects with an internal server error (see logs below).

If I change **only** `destinationAddress` and `gasExcessAddress` to the **user wallet address**, the same quote works:

```ts
const tx = await omniston.buildTransfer({
  quote, // same quote as above
  sourceAddress: {
    address: "<USER_WALLET_ADDRESS>",
    blockchain: Blockchain.TON,
  },
  destinationAddress: {
    address: "<USER_WALLET_ADDRESS>",
    blockchain: Blockchain.TON,
  },
  gasExcessAddress: {
    address: "<USER_WALLET_ADDRESS>",
    blockchain: Blockchain.TON,
  },
  useRecommendedSlippage: false,
});
```

In contrast, for **other resolvers / swap routes** we can successfully call `buildTransfer` with:

```ts
destinationAddress: <CUSTOM_CONTRACT_ADDRESS>,
gasExcessAddress:  <CUSTOM_CONTRACT_ADDRESS>,
```

So the problem seems specific to the combination of:

- **TONCO resolver is used**, and
- using a **custom contract address** as both destination and gasExcess.

---

## Observed error

```txt
[ERROR] buildTransactions: omniston.buildTransfer failed 
{
  index: 0,
  message: "Internal server error: 5: [10558001662133031401601892777664537659284229659435202380369647623999038179300]",
  stack: "Error: Internal server error: 5: [10558001662133031401601892777664537659284229659435202380369647623999038179300]
    at wrapError (...)
    at wrapErrorsAsync (...)
    at async omniston.buildTransfer(...)
    ...",
  quoteSummary: {
    quoteId: "c17b51bd40f738259d0c98a3c1b9871a87d9a75a89898be3bc7524562ccaff8d",
    bidAsset: { blockchain: 607, address: "EQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAM9c" }, // TON
    askAsset: { blockchain: 607, address: "EQC98_qAmNEptUtPc7W6xdHh_ZHrBUFpw5Ft_IzNU20QAJav" }   // tsTON
  }
}
```

- As soon as we change `destinationAddress` / `gasExcessAddress` back to the **user wallet address**, this error disappears for the same `quoteId`.
- For other resolvers / routes, using a **custom address** as destination/excess does **not** produce this error.

---

## Expected behavior

From the Node.js SDK docs:

- `destinationAddress`:  
  > The address on askBlockchain that will receive result of the trade
- `gasExcessAddress`:  
  > The address that will receive the gas not spent by the trade

And from the TypeScript types:

- Both are generic `Address` fields, not restricted to wallet addresses.

So my expectation is that:

- It should be **supported** to send the swap result to a custom contract address, or
- If this is not supported for some routes/resolvers, `buildTransfer` should return a **clear 4xx-style error** (e.g. “unsupported destination address for this route”), instead of an internal server error.

---

## Actual behavior

- For a TON → tsTON quote where the TONCO resolver is used, any attempt to set both `destinationAddress` and `gasExcessAddress` to a custom contract address results in:

  ```txt
  Error: Internal server error: 5: [30986206173307282075924838199273108445273148111914542043079863505487939582408]
  ```

- The same quote works correctly if we send the swap result to the **user wallet address**.
- For other resolvers / routes, we can use a custom destination/excess address without issues.

---

## Questions

1. Is using a custom contract address as `destinationAddress` / `gasExcessAddress` **officially supported** in Omniston for all resolvers, including the TONCO resolver?
2. If yes, could you please investigate and fix the internal server error for the TONCO resolver route with a custom destination/excess address?
3. If not, could you clarify this in the docs and return a more explicit error instead of a 500-style “Internal server error”?

Thank you!
