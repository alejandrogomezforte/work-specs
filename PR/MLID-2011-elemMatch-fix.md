# MLID-2011 — $elemMatch Fix in getCountForEntity()

## Bug: Cross-Element False Positives in Entity Array Queries

### Problem

`getCountForEntity()` in `services/mongodb/notification.ts` used **dot-notation** to query the `entities` array:

```typescript
const matchStage = {
  ...targetFilter,
  'entities.entityType': entityType,
  'entities.entityId': entityId,
};
```

MongoDB evaluates dot-notation conditions on arrays **independently across all elements**, not as a pair on the same element. This causes **cross-element false positives**.

### Concrete Example

A notification for "new document BBB added to order AAA" has:

```typescript
entities: [
  { entityType: 'order', entityId: 'AAA' },
  { entityType: 'document', entityId: 'BBB' },
]
```

Querying for **"all notifications for order BBB"**:

```typescript
{ 'entities.entityType': 'order', 'entities.entityId': 'BBB' }
```

MongoDB evaluates:
- `entities.entityType: 'order'` -- element 0 satisfies this (entityType is 'order')
- `entities.entityId: 'BBB'` -- element 1 satisfies this (entityId is 'BBB')

**Result: FALSE POSITIVE.** Both conditions are satisfied, but by **different** array elements. The notification is linked to order AAA and document BBB, not order BBB.

### Why This Matters for Document Notifications

Every document notification will have exactly two entities: one `order` and one `document`. This makes cross-element false positives **guaranteed** in practice, not a theoretical edge case.

For example, the Order Tracker badge count for an order would include notifications that belong to completely different orders if any of their linked document IDs happen to match an order ID, or vice versa.

### Fix

Use `$elemMatch` to force both conditions to match on the **same** array element:

```typescript
const matchStage = {
  ...targetFilter,
  entities: { $elemMatch: { entityType, entityId } },
};
```

`$elemMatch` guarantees that a single element in the `entities` array has both `entityType` AND `entityId` matching, eliminating cross-element false positives.

### Test Added

```typescript
it('should use $elemMatch to match entity type and id on the same array element', async () => {
  // Asserts the pipeline uses entities: { $elemMatch: { entityType, entityId } }
  // Asserts dot-notation keys are NOT present in the match stage
});
```

### Files Changed

- `apps/web/services/mongodb/notification.ts` -- `getCountForEntity()` match stage
- `apps/web/services/mongodb/notification.test.ts` -- new test for $elemMatch assertion

### References

- [MongoDB $elemMatch query operator](https://www.mongodb.com/docs/manual/reference/operator/query/elemMatch/)
- PR #1049 review comment flagging this bug
