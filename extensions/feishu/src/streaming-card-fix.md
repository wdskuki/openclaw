# Fix for Issue #43704: Feishu streaming card merges unrelated replies

## Problem
When the agent produces multiple independent final replies (`replies=2`) in a single request, the Feishu streaming card incorrectly merges content from the second reply into the first card.

## Root Cause
The `mergeStreamingText` function's fallback behavior concatenates unrelated content when there's no overlap:
```typescript
// Fallback for fragmented partial chunks: append as-is to avoid losing tokens.
return `${previous}${next}`;
```

This is intended for partial streaming chunks, but incorrectly merges unrelated final replies.

## Solution
Track whether the streaming session has already been closed (final delivery sent), and prevent merging new content into a closed session.

## Changes Needed

### 1. Add tracking for final delivery state
In `FeishuStreamingSession` class, add a flag to track if final delivery has occurred.

### 2. Modify close() method
Check if session is already closed before merging new content.

### 3. Add test case
Add test to verify that unrelated final replies are not merged.

## Implementation

```typescript
// Add to FeishuStreamingSession class
private finalDelivered = false;

// Modify close() method
async close(finalText?: string): Promise<void> {
  // If already closed and final was delivered, don't merge new content
  if (!this.state || (this.closed && this.finalDelivered)) {
    return;
  }
  
  // ... rest of close logic ...
  
  this.finalDelivered = true;
}
```

## Alternative Approach
Instead of modifying the session state, we could modify `mergeStreamingText` to have a "strict" mode that refuses to merge completely unrelated content (content with no overlap and significantly different lengths).
