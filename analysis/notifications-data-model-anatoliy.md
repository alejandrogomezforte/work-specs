# Notifications Data Model

## Overview

The notifications system supports real-time alerts for users in the Local Infusion platform. It is designed around two notification types with different read-tracking behaviors, polymorphic entity linking, and flexible targeting.

**Jira Epic:** [MLID-2011](https://localinfusion.atlassian.net/browse/MLID-2011) (Orders: Document Notifications)

## Collections

### `notifications`

Stores all notification records. Each notification has a type, targeting criteria, optional entity links, and lifecycle metadata.

```
{
  _id:              ObjectId
  title:            string          // short display title
  message:          string          // notification body
  type:             "task" | "user" // determines read tracking behavior
  priority:         "low" | "normal" | "high"

  // Targeting — at least one should be set
  targetUserId:     string | null   // specific user
  targetLocationId: string | null   // all users assigned to this location
  targetRole:       string | null   // all users with this role

  // Linked entities — polymorphic, multiple allowed
  entities: [
    { entityType: string, entityId: string }
  ]

  // Read state (task type only — global read)
  isRead:           boolean         // only meaningful for type === "task"
  readBy:           string | null   // userId who acknowledged it
  readAt:           Date | null

  // Lifecycle
  expiresAt:        Date | null     // optional TTL
  createdBy:        string          // userId or "system"
  createdAt:        Date            // auto
  updatedAt:        Date            // auto
}
```

**Indexes:**

| Index | Purpose |
|---|---|
| `{ targetUserId: 1, createdAt: -1 }` | User inbox queries |
| `{ targetLocationId: 1, createdAt: -1 }` | Location inbox queries |
| `{ targetRole: 1, createdAt: -1 }` | Role inbox queries |
| `{ 'entities.entityType': 1, 'entities.entityId': 1 }` | Entity lookup (e.g., "all notifications for order X") |
| `{ expiresAt: 1 }` (sparse) | TTL cleanup for expired notifications |

### `notificationreads`

Per-user read tracking for `user`-type notifications. Each record means "this user has read this notification."

```
{
  _id:              ObjectId
  notificationId:   ObjectId        // ref -> notifications._id
  userId:           string
  readAt:           Date
}
```

**Indexes:**

| Index | Purpose |
|---|---|
| `{ notificationId: 1, userId: 1 }` (unique) | One read record per user per notification |
| `{ userId: 1 }` | "What has this user read" queries |

## Notification Types

### Task Notifications (`type: "task"`)

**Use case:** Actionable items that any team member can acknowledge (e.g., "New document added to order #123"). Once anyone marks it as read, it is read for everyone in the target group.

**Read tracking:** Single `isRead` boolean on the notification document itself. The `readBy` field records who acknowledged it.

**Example:** A new fax arrives for an order. The notification targets all users at the clinic location. The first user to review it clicks "Acknowledge" and the notification clears for the entire team.

### User Notifications (`type: "user"`)

**Use case:** Informational notifications where each user independently tracks their own read state (e.g., "You have been assigned to order #456").

**Read tracking:** Separate records in the `notificationreads` collection. Each user has their own read state — one user marking it read does not affect others.

## Targeting

A notification can target users in three ways. The system matches any user who satisfies **at least one** of the targeting criteria (OR logic).

| Target | Field | Example |
|---|---|---|
| Specific user | `targetUserId` | `"507f1f77bcf86cd799439011"` |
| All users at a location | `targetLocationId` | `"clinic-east"` |
| All users with a role | `targetRole` | `"admin"` |

A user sees a notification if:
- `targetUserId === user.id` OR
- `targetLocationId IN user.locations` OR
- `targetRole === user.role`

**Note:** A user can belong to multiple locations. The service accepts an array of `locationIds` and uses `$in` to match.

## Entity Linking

Notifications can be linked to one or more entities. This enables queries like "show me all notifications for this order" or "how many unread notifications does this patient have?"

```typescript
entities: [
  { entityType: 'order', entityId: '6612a1b2c3d4e5f6a7b8c9d0' },
  { entityType: 'document', entityId: '6612a1b2c3d4e5f6a7b8c9d1' },
]
```

**Supported entity types** (open-ended, no enum constraint):
- `order`
- `document`
- `patient`
- `appointment`
- `clinical-review`
- Any future entity type — no schema changes needed

## Service API

All functions are in `apps/web/services/mongodb/notification.ts`.

### Create a Notification

```typescript
import { createNotification } from '@/services/mongodb/notification';

const notification = await createNotification({
  title: 'New Document Added',
  message: 'A new fax has been added to Order #ORD-1234',
  type: 'task',
  priority: 'high',
  targetLocationId: 'clinic-east',
  entities: [
    { entityType: 'order', entityId: '6612a1b2c3d4e5f6a7b8c9d0' },
    { entityType: 'document', entityId: '6612a1b2c3d4e5f6a7b8c9d1' },
  ],
  createdBy: 'system',
});
```

### Get Notifications for a User

Returns notifications matching the user's ID, locations, or role. Sorted by `createdAt` descending (newest first). Supports pagination.

```typescript
import { getNotificationsForUser } from '@/services/mongodb/notification';

const notifications = await getNotificationsForUser({
  userId: session.user.id,
  locationIds: session.user.userObj?.locations ?? [],
  role: session.user.role,
  limit: 20,
  offset: 0,
});
```

### Get Unread Count (Badge)

Returns the total number of unread notifications for a user across both types.

- **Task notifications:** counts where `isRead === false`
- **User notifications:** counts where no matching `notificationreads` record exists for this user

```typescript
import { getUnreadCountForUser } from '@/services/mongodb/notification';

const unreadCount = await getUnreadCountForUser({
  userId: session.user.id,
  locationIds: session.user.userObj?.locations ?? [],
  role: session.user.role,
});
// Returns: 7
```

### Get Count for an Entity

Count notifications linked to a specific entity. Optionally filter to unread only.

```typescript
import { getCountForEntity } from '@/services/mongodb/notification';

// All notifications for this order
const totalCount = await getCountForEntity({
  entityType: 'order',
  entityId: '6612a1b2c3d4e5f6a7b8c9d0',
  userId: session.user.id,
  locationIds: session.user.userObj?.locations ?? [],
  role: session.user.role,
});

// Only unread notifications for this order
const unreadCount = await getCountForEntity({
  entityType: 'order',
  entityId: '6612a1b2c3d4e5f6a7b8c9d0',
  userId: session.user.id,
  locationIds: session.user.userObj?.locations ?? [],
  role: session.user.role,
  unreadOnly: true,
});
```

### Mark a Task Notification as Read

Global read — marks the notification as read for everyone. Records who acknowledged it.

```typescript
import { markTaskAsRead } from '@/services/mongodb/notification';

const updated = await markTaskAsRead(notificationId, session.user.id);
// updated.isRead === true
// updated.readBy === session.user.id
// updated.readAt === Date
```

### Mark a User Notification as Read

Per-user read — only affects this user's read state.

```typescript
import { markUserNotificationAsRead } from '@/services/mongodb/notification';

await markUserNotificationAsRead(notificationId, session.user.id);
// Silently ignores if already marked as read (duplicate key)
```

### Bulk Acknowledge User Notifications

Mark multiple notifications as read for a user in one operation. Useful for "Acknowledge All" buttons.

```typescript
import { bulkMarkUserNotificationsAsRead } from '@/services/mongodb/notification';

await bulkMarkUserNotificationsAsRead(
  ['notif-id-1', 'notif-id-2', 'notif-id-3'],
  session.user.id
);
// Silently skips any already-read notifications
```

### Check If Read by User

Check whether a specific user has read a specific notification (for `user`-type).

```typescript
import { isReadByUser } from '@/services/mongodb/notification';

const hasRead = await isReadByUser(notificationId, session.user.id);
// Returns: true | false
```

## Integration with SignalR

When a notification is created, the backend can push it to targeted users in real-time via the SignalR service:

```typescript
import { createNotification } from '@/services/mongodb/notification';
import { getSignalRService } from '@/services/signalr';

// 1. Create the notification in MongoDB
const notification = await createNotification({
  title: 'New Document Added',
  message: 'A new fax arrived for Order #ORD-1234',
  type: 'task',
  targetLocationId: 'clinic-east',
  entities: [{ entityType: 'order', entityId: orderId }],
  createdBy: 'system',
});

// 2. Push to connected clients via SignalR
const signalr = await getSignalRService();
const token = signalr.generateServerToken();
const endpoint = signalr.getEndpoint();

await fetch(
  `${endpoint}/api/hubs/li-hub/groups/location-clinic-east/:send?api-version=2022-11-01`,
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify({
      target: 'notification',
      arguments: [notification],
    }),
  }
);
```

## Query Performance

The data model is optimized for the primary query patterns:

| Query | Index Used | Expected Performance |
|---|---|---|
| User inbox (my notifications) | `targetUserId + createdAt` | Fast — indexed scan |
| Location inbox | `targetLocationId + createdAt` | Fast — indexed scan |
| Unread count | Aggregation with `$facet` + `$lookup` | Moderate — join with notificationreads |
| Entity count | `entities.entityType + entities.entityId` | Fast — compound index |
| Check if read | `notificationId + userId` (unique) | Fast — unique index lookup |

**Volume assumptions:** ~100 notifications per location per day. At this volume, the aggregation pipelines for unread counts perform well without additional optimization.
