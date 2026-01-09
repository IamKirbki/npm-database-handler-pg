# @kirbkis/database-handler-pg

PostgreSQL adapter for Kirbkis Database Handler.

## Installation

```bash
npm install @iamkirbki/database-handler-core
npm install @iamkirbki/database-handler-pg
```

## Features

- ✅ PostgreSQL support via node-postgres (pg)
- ⚡ Connection pooling
- 🔒 Type-safe with TypeScript
- 📝 Full schema builder support
- 🎯 Unified API with other adapters

## Quick Start

```typescript
import { PostgresAdapter } from '@iamkirbki/database-handler-pg';
import { Container } from '@iamkirbki/database-handler-core';

// Connect to database
const db = new PostgresAdapter();
await db.connect({
    host: 'localhost',
    port: 5432,
    database: 'myapp',
    user: 'postgres',
    password: 'secret',
    max: 20  // Connection pool size
});

Container.RegisterAdapter(db);

// Create a table
await db.createTable('users', (table) => {
    table.integer('id').primaryKey().increments();
    table.string('name', 100);
    table.string('email', 100).unique();
    table.timestamps();
});

// Use core classes
import { Table } from '@iamkirbki/database-handler-core';
const usersTable = new Table('users');
const users = await usersTable.Records<User>();
```

## Adapter API

### Constructor

```typescript
const db = new PostgresAdapter();
```

### Connection

```typescript
await db.connect(config: PoolConfig): Promise<void>
```

**PoolConfig Options:**
```typescript
{
    host: string;
    port: number;
    database: string;
    user: string;
    password: string;
    max?: number;                    // Max pool size (default: 10)
    idleTimeoutMillis?: number;      // Idle timeout (default: 30000)
    connectionTimeoutMillis?: number; // Connection timeout (default: 2000)
    ssl?: boolean | object;          // SSL config
}
```

### Methods

```typescript
// Schema operations
await db.createTable(name: string, callback: (table) => void): Promise<void>
await db.alterTable(name: string, callback: (table) => void): Promise<void>
await db.dropTable(name: string): Promise<void>

// Execute queries
await db.execute(sql: string, params?: any): Promise<any>
await db.query(sql: string, params?: any): Promise<any[]>

// Connection management
await db.disconnect(): Promise<void>
```

## Schema Builder

PostgreSQL-specific data types and constraints:

```typescript
await db.createTable('posts', (table) => {
    // Serial primary key
    table.integer('id').primaryKey().increments();
    
    // String types
    table.string('title', 200);      // VARCHAR(200)
    table.text('content');           // TEXT
    table.uuid('uuid');              // UUID (native)
    
    // Numbers
    table.integer('views');          // INTEGER
    table.decimal('price', 10, 2);   // DECIMAL(10,2)
    table.float('rating');           // REAL
    
    // Other types
    table.boolean('is_active');      // BOOLEAN
    table.json('metadata');          // JSONB (native)
    table.timestamp('created_at');   // TIMESTAMP
    table.time('start_time');        // TIME
    
    // Constraints
    table.string('email').unique();
    table.string('bio').nullable();
    table.integer('status').defaultTo(1);
    table.integer('user_id').foreignKey('users', 'id');
    
    // Helpers
    table.timestamps();              // created_at, updated_at
    table.softDeletes();            // deleted_at
    table.morphs('commentable');    // commentable_id, commentable_type
});
```

## PostgreSQL Specifics

### Data Type Mapping

| Method | PostgreSQL Type | Notes |
|--------|-----------------|-------|
| `string()` | VARCHAR | Optional length parameter |
| `text()` | TEXT | For large text |
| `integer()` | INTEGER | Whole numbers |
| `decimal()` | DECIMAL | Precise decimals |
| `float()` | REAL | Floating point |
| `boolean()` | BOOLEAN | True/false |
| `json()` | JSONB | Native JSON with indexing |
| `uuid()` | UUID | Native UUID type |
| `timestamp()` | TIMESTAMP | Date and time |
| `time()` | TIME | Time only |

### Serial/Auto-Increment

```typescript
table.integer('id').primaryKey().increments();
// Generates: id SERIAL PRIMARY KEY
```

### Foreign Keys

```typescript
table.integer('user_id').foreignKey('users', 'id');
// Generates: user_id INTEGER REFERENCES users(id)
```

### Enum Values

```typescript
table.enum('status', ['draft', 'published', 'archived']);
// Generates: status TEXT CHECK (status IN ('draft', 'published', 'archived'))
```

## Examples

### Basic Connection

```typescript
const db = new PostgresAdapter();
await db.connect({
    host: 'localhost',
    port: 5432,
    database: 'myapp',
    user: 'postgres',
    password: 'secret'
});
Container.RegisterAdapter(db);
```

### Connection Pooling

```typescript
const db = new PostgresAdapter();
await db.connect({
    host: 'localhost',
    port: 5432,
    database: 'myapp',
    user: 'postgres',
    password: 'secret',
    max: 20,                    // Pool size
    idleTimeoutMillis: 30000,   // 30 seconds
    connectionTimeoutMillis: 2000
});
```

### SSL Connection

```typescript
const db = new PostgresAdapter();
await db.connect({
    host: 'db.example.com',
    port: 5432,
    database: 'myapp',
    user: 'postgres',
    password: 'secret',
    ssl: {
        rejectUnauthorized: false
    }
});
```

### Multiple Databases

```typescript
const mainDb = new PostgresAdapter();
await mainDb.connect(mainConfig);

const analyticsDb = new PostgresAdapter();
await analyticsDb.connect(analyticsConfig);

Container.RegisterAdapter(mainDb, 'main', true);  // Default
Container.RegisterAdapter(analyticsDb, 'analytics');

// Use specific database
const table = new Table('events', 'analytics');
```

### Complete CRUD

```typescript
import { PostgresAdapter } from '@iamkirbki/database-handler-pg';
import { Container, Table, Record } from '@iamkirbki/database-handler-core';

const db = new PostgresAdapter();
await db.connect({
    host: 'localhost',
    port: 5432,
    database: 'myapp',
    user: 'postgres',
    password: 'secret'
});
Container.RegisterAdapter(db);

await db.createTable('users', (table) => {
    table.integer('id').primaryKey().increments();
    table.uuid('uuid').unique();
    table.string('name', 100);
    table.string('email', 100).unique();
    table.boolean('is_active').defaultTo(true);
    table.json('preferences').nullable();
    table.timestamps();
});

const usersTable = new Table('users');

// Create
const user = new Record<User>('users', {
    uuid: crypto.randomUUID(),
    name: 'Alice',
    email: 'alice@example.com',
    preferences: { theme: 'dark' }
});
await user.Insert();

// Read
const users = await usersTable.Records<User>({ 
    where: { is_active: true } 
});

// Update
const alice = await usersTable.Record<User>({ 
    where: { email: 'alice@example.com' } 
});
if (alice) {
    alice.values.name = 'Alice Smith';
    alice.values.preferences = { theme: 'light' };
    await alice.Update();
}

// Delete
await alice.Delete();
```

### Transactions

```typescript
// PostgreSQL adapter handles transactions internally
// Each query runs in its own transaction by default
```

### JSON/JSONB Queries

```typescript
import { Query } from '@iamkirbki/database-handler-core';

// Query JSONB columns
const query = new Query({
    tableName: 'users',
    query: "SELECT * FROM users WHERE preferences->>'theme' = @theme",
    parameters: { theme: 'dark' }
});

const users = await query.All<User>();
```

## Performance Tips

1. **Connection Pooling**: Configure appropriate pool size
2. **Indexes**: Create indexes on frequently queried columns
3. **JSONB**: Use JSONB for JSON data (faster than TEXT)
4. **Prepared Statements**: Automatically used for all queries
5. **Connection Limits**: Monitor and adjust `max` pool size

```typescript
// Example: Add indexes
await db.execute(`
    CREATE INDEX idx_users_email ON users(email);
    CREATE INDEX idx_users_status ON users(status);
`);
```

## Connection Management

```typescript
// Graceful shutdown
process.on('SIGINT', async () => {
    await db.disconnect();
    process.exit(0);
});
```

## Environment Variables

```bash
# .env file
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=myapp
POSTGRES_USER=postgres
POSTGRES_PASSWORD=secret
POSTGRES_MAX_CONNECTIONS=20
```

```typescript
import 'dotenv/config';

const db = new PostgresAdapter();
await db.connect({
    host: process.env.POSTGRES_HOST,
    port: parseInt(process.env.POSTGRES_PORT),
    database: process.env.POSTGRES_DB,
    user: process.env.POSTGRES_USER,
    password: process.env.POSTGRES_PASSWORD,
    max: parseInt(process.env.POSTGRES_MAX_CONNECTIONS)
});
```

## Documentation

- [Core Documentation](../core/README.md)
- [Query Guide](../core/src/base/Wiki/Query.md)
- [Table Guide](../core/src/base/Wiki/Table.md)
- [Record Guide](../core/src/base/Wiki/Record.md)
- [Schema Builder](../core/src/abstract/Wiki/SchemaTableBuilder.md)

## License

ISC License
