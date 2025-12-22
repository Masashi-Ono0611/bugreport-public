# Omniston SDK Stream Consumer Closure Issue

## Summary

- **Issue**: A closure in the API client's method registration was capturing an outdated reference to a `Map` of stream consumers. This caused only the last registered RFQ subscription to receive updates, as all handlers were using the same stale `Map` reference.
- **Location**: `packages/omniston-sdk/src/ApiClient/ApiClient.ts` in the handler registration logic.
- **Impact**: Multiple RFQ subscriptions could not function correctly simultaneously, as all but the most recent subscription would fail to receive updates.

## Environment

- **SDK**: `@bagel/omniston-sdk`
- **Version**: Version: @ston-fi/omniston-sdk@0.7.6 (and corresponding omniston-sdk-react version)
- **Environment**: Any environment using multiple RFQ subscriptions simultaneously

## Minimal Reproduction

1. Set up multiple RFQ subscriptions using the Omniston SDK
2. Observe that only the last subscription receives updates
3. Check the network requests to confirm that while all subscriptions are established, only the most recent one triggers handler callbacks

## Observed Behavior

```typescript
// Buggy implementation
this.serverAndClient.addMethod(method, (payload) => {
  const consumerMap = result; // Old map from closure
  const consumer = consumerMap?.get(payload.subscription);
  // consumer is undefined for all but the most recent subscription
});
```

## Expected Behavior

Each subscription handler should be able to look up its consumer from the current state of `streamConsumers`.

## Solution

The fix involves always reading the latest consumer map from the instance property rather than capturing it in the closure:

```typescript
// Fixed implementation
this.serverAndClient.addMethod(method, (payload) => {
  const consumerMap = this.streamConsumers.get(method);
  const consumer = consumerMap?.get(payload.subscription);
  // Now correctly finds the consumer for each subscription
});
```

## Technical Details

- **Root Cause**: The closure was capturing the value of `result` at the time of method registration, rather than looking up the current value on each invocation.
- **Fix Impact**: Low risk - only changes how the consumer map is accessed, not the data structure or behavior.
- **Testing**: Should be verified by testing multiple concurrent RFQ subscriptions to ensure all receive their respective updates.

## Additional Notes

- This issue was particularly problematic in scenarios requiring multiple concurrent RFQ subscriptions, such as real-time price updates for multiple trading pairs.
- The fix maintains backward compatibility while resolving the subscription update issue.

---

*This document was generated to document the bug and its resolution for future reference.*
