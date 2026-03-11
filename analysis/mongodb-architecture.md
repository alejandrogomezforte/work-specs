# MongoDB Architecture Analysis — li-call-processor

> **Repo:** li-call-processor
> **Date:** 2026-01-30
> **Next.js Version:** 14 (App Router)

---

## 1. Libraries

The project uses **two MongoDB libraries** side by side:

| Library              | Version | Purpose                                   |
| -------------------- | ------- | ----------------------------------------- |
| **Mongoose ODM**     | 8.9.5   | Schema-based model operations (newer code) |
| **MongoDB Native Driver** | 6.3.0 | Direct collection queries (legacy code)   |

This is a **hybrid approach** — the codebase is gradually migrating from the native driver to Mongoose. A TODO in `models/Patient.ts` references ticket `MLID-128` to complete the migration.

---

## 2. Connection Architecture

**File:** `services/mongodb/connect.ts` (296 lines)

The connection module implements a **dual-connection singleton pattern** designed for Next.js serverless environments.

### 2.1 Dual Connections

Two separate connections are established:

1. **MongoDB Native Client** — for direct `db.collection()` queries used by legacy service files.
2. **Mongoose Connection** — for schema-based model operations used by newer service files.

### 2.2 Global Caching (Singleton)

Connections are stored on `globalThis` to survive across serverless function invocations and prevent connection exhaustion:

```typescript
declare global {
  var _mongoClientPromise: Promise<MongoClient> | undefined;
  var _mongoosePromise: Promise<Mongoose> | undefined;
  var _mongoClient: MongoClient | undefined;
  var _mongooseConnection: Mongoose | undefined;
}
```

### 2.3 Connection Pool Configuration

```typescript
serverSelectionTimeoutMS: 10000
connectTimeoutMS: 10000
socketTimeoutMS: 360000       // 6 minutes for long-running queries
maxPoolSize: 10
minPoolSize: 2
maxIdleTimeMS: 60000          // Close idle connections after 60s
retryWrites: true
retryReads: true
w: 'majority'
compressors: ['zlib']         // Network compression enabled
```

Mongoose-specific settings:

```typescript
mongoose.set('strictQuery', false);
mongoose.set('bufferCommands', false);
```

### 2.4 Build-Phase Handling

Connections are skipped during `next build` by checking the `NEXT_PHASE` environment variable. This prevents connection exhaustion during the build process.

### 2.5 Exported Functions

| Function                  | Returns                        | Purpose                              |
| ------------------------- | ------------------------------ | ------------------------------------ |
| `connectToDatabase()`     | `{ db: Db, client: MongoClient }` | Native driver access                 |
| `getMongooseConnection()` | `Mongoose`                     | Mongoose instance for model operations |
| `closeConnection()`       | `void`                         | Graceful shutdown (used in tests)    |

### 2.6 Error Handling

Event listeners monitor `close` and `error` events on both connections. On failure, the global cache is cleared so the next invocation establishes a fresh connection.

---

## 3. Service Layer Architecture

The architecture follows a **DAO (Data Access Object)** pattern:

```
API Routes / Pages
       |
  Service Layer    (services/mongodb/*.ts)
       |
  Models/Schemas   (models/*.ts)
       |
  Connection       (services/mongodb/connect.ts)
       |
  MongoDB
```

### 3.1 Service Layer (`services/mongodb/`)

30+ service files, each responsible for a domain. Two flavors exist depending on which driver they use:

**Native driver services (older):**
- Call `connectToDatabase()`, then `db.collection<Type>(COLLECTION.X).find(...)`
- Examples: `patient.ts`, `callHistory.ts`, `patientInsurance.ts`

**Mongoose services (newer):**
- Use model methods directly: `Order.findById().lean()`
- Examples: `orders.ts`, `chat.ts`, `contentLibrary.ts`

The public API is re-exported from `services/mongodb/index.ts`:

```typescript
export * as callHistory from './callHistory';
export * as callInProgress from './callInProgress';
export * as chat from './chat';
export * as dictionaries from './dictionaries';
export * as financialCalculation from './financialCalculation';
export * as integrations from './integrations';
export * as patient from './patient';
export * as patientInsurance from './patientInsurance';
```

### 3.2 Data Access Pattern Comparison

| Aspect          | Native MongoDB Client                          | Mongoose ORM                               |
| --------------- | ---------------------------------------------- | ------------------------------------------ |
| **Connection**  | `connectToDatabase()` then `db.collection()`   | Mongoose model methods directly            |
| **Queries**     | `.find()`, `.findOne()`, `.insertOne()`, `.updateOne()` | `.findById()`, `.create()`, `.findOneAndUpdate()` |
| **Performance** | `.toArray()` returns documents                 | `.lean()` for plain objects                |
| **Transactions**| Manual session handling                        | `.startSession()` with `.withTransaction()` |
| **Indexes**     | Manual creation                                | Schema-defined indexes                     |

### 3.3 Representative Service Examples

**Patient Service (`patient.ts`)** — Native Client:
- Functions: `getPatients()`, `insertPatient()`, `deletePatientById()`, `searchExtendedPatients()`
- Uses `db.collection<Type>(COLLECTION.Patients).find(...)`

**Orders Service (`orders.ts`)** — Mongoose:
- Functions: `getOrderById()`, `getOrderByWeInfuseId()`, `getOrdersByPatientId()`, `createOrder()`, `updateOrder()`
- Uses `Order.findById().lean()`, `Order.create()`, `Order.findOneAndUpdate()`

**Chat Service (`chat.ts`)** — Mongoose with Transactions:
- Implements `saveConversationTurn()` with MongoDB transactions for atomicity
- Uses session-based transactions for multi-document operations

**Call History Service (`callHistory.ts`)** — Native Client:
- Uses MongoDB positional operator (`$`) for nested array updates within patient documents

---

## 4. Models & Schemas

**Location:** `models/` (30+ model files)

Each model follows a consistent pattern:

```typescript
import mongoose, { Document, Schema } from 'mongoose';

// 1. TypeScript interface
export interface IModelName extends Document {
  field: string;
  // ...
}

// 2. Schema definition with indexes
const ModelNameSchema = new Schema<IModelName>({
  field: { type: String, required: true },
  // ...
});

ModelNameSchema.index({ field1: 1, field2: 1 });
ModelNameSchema.index({ field: 'text' });

// 3. Hot-reload-safe model creation
export const ModelName =
  (mongoose.models.ModelName as mongoose.Model<IModelName>) ||
  mongoose.model<IModelName>('ModelName', ModelNameSchema);
```

### Key Models

| Model              | File              | Notable Features                                    |
| ------------------ | ----------------- | --------------------------------------------------- |
| **Order**          | `Order.ts`        | WeInfuse IDs, compound index on `patientId + active` |
| **ChatMessage**    | `ChatMessage.ts`  | Text index on `content`, compound on `threadId + messageSeq` |
| **Configuration**  | `Configuration.ts`| Nested schema for `ConfigurationValues`             |
| **Patient**        | `Patient.ts`      | Legacy model, mixed native + Mongoose usage         |

---

## 5. Collection Name Registry

**File:** `utils/constants/collections.ts` (33 lines)

A single `COLLECTION` constant maps logical names to actual MongoDB collection names, used by native driver services to avoid hardcoded strings:

```typescript
export const COLLECTION = {
  Patients: 'patients',
  Providers: 'cmsnpiproviders',
  Configurations: 'configurations',
  Appointments: 'appointments',
  Insurances: 'insurances',
  PatientInsurances: 'patient_insurances',
  PatientLocations: 'patientLocations',
  UsersWhiteList: 'usersWhiteList',
  Dictionaries: 'dictionaries',
  Locks: 'locks',
  Users: 'users',
  OrdersCounter: 'orderscounter',
  NewOrders: 'neworders',
  MaintenanceOrders: 'maintenanceorders',
  LeadOrders: 'leadorders',
  OrderProtocols: 'orderprotocols',
  Conversations: 'conversations',
  ConversationItems: 'conversationItems',
  Messages: 'messages',
  Drugs: 'drugs',
  LIDrugs: 'lidrugs',
  LIBiosimilars: 'libiosimilars',
  DrugRequirements: 'drugRequirements',
  DrugSettings: 'drugSettings',
  DrugConfigurations: 'drugConfigurations',
  Orders: 'orders',
  ClinicalReviews: 'clinicalReviews',
  Documents: 'documents',
  Intakes: 'intakes',
  InsurancePlans: 'insurance_plans',
};
```

---

## 6. Performance Optimizations

1. **Connection Pooling** — Min 2, max 10 connections per instance.
2. **Network Compression** — zlib enabled to reduce payload size.
3. **Lean Queries** — Mongoose `.lean()` returns plain objects instead of full Mongoose documents.
4. **Indexes** — Single-field, compound, text, and unique indexes defined at the schema level.
5. **Idle Connection Management** — Connections closed after 60 seconds of inactivity.
6. **Query Timeouts** — 10-second connection timeout, 6-minute socket timeout for long queries.
7. **Retry Logic** — Automatic `retryWrites` and `retryReads` enabled.

---

## 7. Testing Support

- **In-memory MongoDB:** Uses `mongodb-memory-server` for test isolation.
- **Graceful shutdown:** `closeConnection()` is called in test teardown.
- **Integration tests:** `npm run test:integration` runs against in-memory MongoDB.
- **Global setup:** `jest.mongodb.global-setup.ts` configures the test database.

---

## 8. File Structure Summary

```
services/mongodb/
  connect.ts              -- Connection management (296 lines)
  index.ts                -- Public API re-exports
  patient.ts              -- Patient data operations (native client)
  orders.ts               -- Order data operations (Mongoose)
  callHistory.ts          -- Call tracking (native client)
  chat.ts                 -- Chat operations (Mongoose + transactions)
  patientInsurance.ts     -- Insurance data (native client)
  [20+ other service files]

models/
  Patient.ts              -- Patient schema (legacy)
  Order.ts                -- Order schema (Mongoose)
  ChatMessage.ts          -- Chat message schema
  Configuration.ts        -- Configuration schema
  [25+ other model files]

utils/constants/
  collections.ts          -- Collection name registry
```

---

## 9. Technical Debt & Migration Notes

| Item | Detail |
| ---- | ------ |
| **Dual-driver usage** | Native driver and Mongoose coexist; tracked under `MLID-128` for full migration to Mongoose. |
| **Inconsistent patterns** | Some services use `connectToDatabase()` (native), others use Mongoose models directly. |
| **Post-migration cleanup** | Once migration completes, `connectToDatabase()`, the native client global cache, and the `COLLECTION` constant can be removed. |
| **Transaction adoption** | Only `chat.ts` uses transactions; other multi-document operations could benefit from them. |
