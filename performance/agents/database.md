---
description: "[Internal] Database agent - use /performance-scan instead"
disable-model-invocation: true
allowed-tools: Read, Glob, Grep
---

# Database Agent (PERF03)

Scan for performance issues related to:
- **N+1 Queries** - Multiple queries where one would suffice
- **Missing Indexes** - Queries on unindexed columns
- **Unbounded Queries** - Queries without LIMIT
- **SELECT * Usage** - Fetching unnecessary columns
- **Unnecessary JOINs** - Over-fetching related data
- **Connection Pooling** - Missing or misconfigured pools
- **Transaction Scope** - Long-running or unnecessary transactions
- **Eager Loading Issues** - Loading relations not needed
- **Missing Query Caching** - Repeated identical queries

---

## 1. N+1 Query Detection

### 1.1 Loop Database Calls

**Search:**
```
Grep: for\s*\(.*\)\s*\{[^}]*(findOne|findById|findByPk|query|execute|select|get\()|\.forEach\([^)]*\)[^}]*(findOne|find|get|query)
Glob: **/*.ts, **/*.js, **/*.py, **/*.go, **/*.java, **/*.php, **/*.cs
```

**Check for:**
- [ ] Database calls inside loops
- [ ] Related entity fetched per parent item
- [ ] Missing eager loading/includes

**N+1 Pattern (TypeORM):**
```typescript
// BAD: N+1 queries - 1 for posts, N for authors
async function getPostsWithAuthors() {
  const posts = await postRepository.find();  // 1 query

  for (const post of posts) {
    post.author = await userRepository.findOne({
      where: { id: post.authorId }
    });  // N queries!
  }

  return posts;
}
```

**Optimized Pattern:**
```typescript
// GOOD: Single query with join
async function getPostsWithAuthors() {
  return postRepository.find({
    relations: ['author']  // Eager load
  });
}

// OR: Using QueryBuilder
async function getPostsWithAuthors() {
  return postRepository
    .createQueryBuilder('post')
    .leftJoinAndSelect('post.author', 'author')
    .getMany();
}
```

**N+1 Pattern (Prisma):**
```typescript
// BAD: N+1 with Prisma
async function getOrdersWithItems() {
  const orders = await prisma.order.findMany();

  for (const order of orders) {
    order.items = await prisma.orderItem.findMany({
      where: { orderId: order.id }
    });
  }
}
```

**Optimized Pattern:**
```typescript
// GOOD: Include relation
async function getOrdersWithItems() {
  return prisma.order.findMany({
    include: { items: true }
  });
}
```

**N+1 Pattern (Django):**
```python
# BAD: N+1 queries
def get_articles_with_authors():
    articles = Article.objects.all()  # 1 query
    for article in articles:
        print(article.author.name)  # N queries!
```

**Optimized Pattern:**
```python
# GOOD: select_related for ForeignKey
def get_articles_with_authors():
    articles = Article.objects.select_related('author').all()
    for article in articles:
        print(article.author.name)  # No additional query

# GOOD: prefetch_related for ManyToMany
def get_articles_with_tags():
    articles = Article.objects.prefetch_related('tags').all()
```

**N+1 Pattern (Sequelize):**
```javascript
// BAD: N+1 queries
const users = await User.findAll();
for (const user of users) {
  user.posts = await user.getPosts();  // N queries
}
```

**Optimized Pattern:**
```javascript
// GOOD: Eager loading
const users = await User.findAll({
  include: [{ model: Post }]
});
```

### 1.2 GraphQL N+1

**Search:**
```
Grep: @ResolveField|@FieldResolver|dataloader|DataLoader
Glob: **/*.resolver.ts, **/*.ts
```

**Check for:**
- [ ] DataLoader usage for batching
- [ ] Parent data included in context
- [ ] Batch resolvers for relations

**N+1 Pattern (NestJS GraphQL):**
```typescript
// BAD: N+1 in field resolver
@ResolveField()
async author(@Parent() post: Post) {
  return this.userService.findById(post.authorId);  // Called per post
}
```

**Optimized Pattern:**
```typescript
// GOOD: Using DataLoader
@ResolveField()
async author(@Parent() post: Post, @Context() ctx) {
  return ctx.userLoader.load(post.authorId);  // Batched
}

// DataLoader setup
const userLoader = new DataLoader(async (ids: string[]) => {
  const users = await this.userService.findByIds(ids);
  return ids.map(id => users.find(u => u.id === id));
});
```

---

## 2. Missing Indexes

### 2.1 Query Patterns Without Indexes

**Search:**
```
Grep: findOne\(.*where|findMany\(.*where|\.where\(|WHERE|filter\(|ORDER BY|GROUP BY
Glob: **/*.ts, **/*.js, **/*.py, **/*.java, **/*.php
```

**Check for indexed columns on:**
- [ ] WHERE clause columns
- [ ] JOIN condition columns
- [ ] ORDER BY columns
- [ ] GROUP BY columns
- [ ] Unique constraint columns

**Search for entity/model definitions:**
```
Glob: **/*.entity.ts, **/models/*.ts, **/entities/*.ts, **/schema.prisma, **/models.py
Grep: @Index|@Column|index:|indexes|db_index
```

**Check against queries:**

**Missing Index Pattern (TypeORM):**
```typescript
// Entity without index
@Entity()
export class User {
  @Column()
  email: string;  // No index!

  @Column()
  status: string;  // No index!
}

// Query using unindexed column
userRepository.find({ where: { email: 'test@example.com' } });
userRepository.find({ where: { status: 'active' } });
```

**Optimized Pattern:**
```typescript
// Entity with indexes
@Entity()
@Index(['email'], { unique: true })
@Index(['status'])
export class User {
  @Column()
  email: string;

  @Column()
  status: string;
}
```

**Missing Index Pattern (Prisma):**
```prisma
// BAD: No index on frequently queried field
model Order {
  id        String   @id
  status    String   // Queried frequently, no index
  createdAt DateTime // Sorted frequently, no index
}
```

**Optimized Pattern:**
```prisma
// GOOD: Indexed fields
model Order {
  id        String   @id
  status    String
  createdAt DateTime

  @@index([status])
  @@index([createdAt])
}
```

**Missing Index Pattern (Django):**
```python
# BAD: No index
class Article(models.Model):
    slug = models.CharField(max_length=200)  # Queried often, no index
    published_date = models.DateTimeField()
```

**Optimized Pattern:**
```python
# GOOD: With indexes
class Article(models.Model):
    slug = models.CharField(max_length=200, db_index=True)
    published_date = models.DateTimeField(db_index=True)

    class Meta:
        indexes = [
            models.Index(fields=['slug', 'published_date']),  # Composite
        ]
```

### 2.2 Composite Index Opportunities

**Search for queries with multiple conditions:**
```
Grep: where:.*\{[^}]*,[^}]*\}|\.where\(.*\.andWhere\(|AND.*WHERE|WHERE.*AND
```

**Check for:**
- [ ] Multi-column WHERE clauses need composite indexes
- [ ] Index column order matches query patterns

---

## 3. Unbounded Queries

### 3.1 Missing LIMIT/Pagination

**Search:**
```
Grep: findAll|findMany|\.find\(\)|\.all\(\)|SELECT.*FROM(?!.*LIMIT)|objects\.all\(\)|\.list\(\)
Glob: **/*.ts, **/*.js, **/*.py, **/*.java, **/*.php
```

**Check for:**
- [ ] All list queries have limits
- [ ] Pagination implemented for user-facing lists
- [ ] Background jobs process in batches

**Unbounded Pattern:**
```typescript
// BAD: No limit - could return millions of rows
async function getAllUsers() {
  return userRepository.find();
}

// BAD: User-controlled limit without maximum
async function getUsers(limit?: number) {
  return userRepository.find({ take: limit });  // limit could be 1000000
}
```

**Optimized Pattern:**
```typescript
// GOOD: Default and maximum limits
const DEFAULT_LIMIT = 20;
const MAX_LIMIT = 100;

async function getUsers(page = 1, limit = DEFAULT_LIMIT) {
  const safeLimit = Math.min(limit, MAX_LIMIT);
  return userRepository.find({
    take: safeLimit,
    skip: (page - 1) * safeLimit
  });
}
```

**Django Pattern:**
```python
# BAD: Unbounded
def get_all_articles():
    return Article.objects.all()

# GOOD: Paginated
from django.core.paginator import Paginator

def get_articles(page=1, per_page=20):
    articles = Article.objects.all()
    paginator = Paginator(articles, min(per_page, 100))
    return paginator.page(page)
```

---

## 4. SELECT * / Over-fetching

### 4.1 Fetching All Columns

**Search:**
```
Grep: SELECT\s+\*|findOne\(\)|findMany\(\)|\.find\(\{|\.get\(|objects\.get\(
Glob: **/*.ts, **/*.js, **/*.py, **/*.sql
```

**Check for:**
- [ ] Only needed columns selected
- [ ] Large text/blob columns excluded when not needed
- [ ] Relation data selected appropriately

**Over-fetch Pattern:**
```typescript
// BAD: Fetches all columns including large ones
async function getUserEmails() {
  const users = await userRepository.find();  // Gets all columns
  return users.map(u => u.email);
}
```

**Optimized Pattern:**
```typescript
// GOOD: Select only needed columns
async function getUserEmails() {
  const users = await userRepository.find({
    select: ['id', 'email']
  });
  return users.map(u => u.email);
}

// OR: Using QueryBuilder
async function getUserEmails() {
  return userRepository
    .createQueryBuilder('user')
    .select(['user.id', 'user.email'])
    .getMany();
}
```

**Prisma Pattern:**
```typescript
// BAD: Fetches all fields
const users = await prisma.user.findMany();

// GOOD: Select specific fields
const users = await prisma.user.findMany({
  select: { id: true, email: true }
});
```

**Django Pattern:**
```python
# BAD: Fetches all fields
users = User.objects.all()

# GOOD: Select specific fields
users = User.objects.only('id', 'email')
# OR
users = User.objects.values('id', 'email')
```

### 4.2 Unnecessary Relation Loading

**Search:**
```
Grep: relations:|include:|select_related|prefetch_related|\.Include\(|\.ThenInclude\(|eager|join
```

**Check for:**
- [ ] Relations loaded only when needed
- [ ] Nested relations limited appropriately
- [ ] No circular loading

**Over-fetch Pattern:**
```typescript
// BAD: Loading deep nested relations
const users = await userRepository.find({
  relations: ['posts', 'posts.comments', 'posts.comments.author', 'profile', 'settings']
});
// Only needed: users with post count
```

**Optimized Pattern:**
```typescript
// GOOD: Load only what's needed
const users = await userRepository
  .createQueryBuilder('user')
  .loadRelationCountAndMap('user.postCount', 'user.posts')
  .getMany();
```

---

## 5. Connection Pooling

### 5.1 Pool Configuration

**Search:**
```
Grep: createConnection|createPool|Pool|pool:|poolSize|max:|min:|connectionLimit|maxPoolSize
Glob: **/*.ts, **/*.js, **/database*.ts, **/db*.ts, **/config*.ts, **/*.py, **/*.go
```

**Check for:**
- [ ] Connection pool configured
- [ ] Pool size appropriate for workload
- [ ] Idle timeout configured
- [ ] Connection validation enabled

**Missing Pool Pattern (Node.js):**
```typescript
// BAD: New connection per query
async function query(sql: string) {
  const connection = await mysql.createConnection(config);
  const result = await connection.query(sql);
  await connection.end();
  return result;
}
```

**Optimized Pattern:**
```typescript
// GOOD: Connection pool
const pool = mysql.createPool({
  host: 'localhost',
  user: 'root',
  database: 'mydb',
  connectionLimit: 10,
  waitForConnections: true,
  queueLimit: 0
});

async function query(sql: string) {
  const [rows] = await pool.query(sql);
  return rows;
}
```

**TypeORM Pool:**
```typescript
// GOOD: TypeORM with pool settings
const dataSource = new DataSource({
  type: 'postgres',
  host: 'localhost',
  port: 5432,
  extra: {
    max: 20,  // Pool size
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000
  }
});
```

**Django Pool (django-db-connection-pool):**
```python
DATABASES = {
    'default': {
        'ENGINE': 'dj_db_conn_pool.backends.postgresql',
        'POOL_OPTIONS': {
            'POOL_SIZE': 10,
            'MAX_OVERFLOW': 10,
            'RECYCLE': 300,
        }
    }
}
```

### 5.2 Connection Leaks

**Search:**
```
Grep: getConnection|acquire|borrow|checkout
```

**Check for:**
- [ ] Connections released after use
- [ ] Try/finally for cleanup
- [ ] Using connection managers

**Leak Pattern:**
```typescript
// BAD: Connection not released on error
async function processData() {
  const conn = await pool.getConnection();
  const data = await conn.query('SELECT * FROM users');
  // If error occurs here, connection leaks
  await processResults(data);
  conn.release();
}
```

**Fixed Pattern:**
```typescript
// GOOD: Using try/finally
async function processData() {
  const conn = await pool.getConnection();
  try {
    const data = await conn.query('SELECT * FROM users');
    await processResults(data);
  } finally {
    conn.release();
  }
}

// BETTER: Using callback pattern
async function processData() {
  return pool.query('SELECT * FROM users')
    .then(data => processResults(data));
}
```

---

## 6. Transaction Scope

### 6.1 Long-Running Transactions

**Search:**
```
Grep: @Transaction|BEGIN|START TRANSACTION|transaction\(|\.transaction\(|with\s+transaction
Glob: **/*.ts, **/*.js, **/*.py, **/*.java
```

**Check for:**
- [ ] Transactions as short as possible
- [ ] No network calls inside transactions
- [ ] No user input waits inside transactions

**Slow Pattern:**
```typescript
// BAD: Long transaction with external call
await dataSource.transaction(async (manager) => {
  const user = await manager.findOne(User, { where: { id } });
  const externalData = await fetchFromApi(user.externalId);  // Network!
  user.data = externalData;
  await manager.save(user);
});
```

**Optimized Pattern:**
```typescript
// GOOD: External call outside transaction
const user = await userRepository.findOne({ where: { id } });
const externalData = await fetchFromApi(user.externalId);

await dataSource.transaction(async (manager) => {
  user.data = externalData;
  await manager.save(user);
});
```

### 6.2 Unnecessary Transactions

**Search:**
```
Grep: @Transactional|@Transaction|transaction\(
```

**Check for:**
- [ ] Single read operations don't need transactions
- [ ] Single write operations may not need explicit transaction
- [ ] Read-only transactions for reads

**Over-use Pattern:**
```typescript
// BAD: Transaction for single read
async function getUser(id: string) {
  return dataSource.transaction(async (manager) => {
    return manager.findOne(User, { where: { id } });
  });
}
```

**Optimized Pattern:**
```typescript
// GOOD: No transaction for single read
async function getUser(id: string) {
  return userRepository.findOne({ where: { id } });
}
```

---

## 7. Query Caching

### 7.1 Repeated Identical Queries

**Search:**
```
Grep: cache:|@Cacheable|\.cache\(|cacheTime|redis\.get|memcache
Glob: **/*.ts, **/*.js, **/*.py, **/*.java
```

**Check for:**
- [ ] Frequently accessed static data is cached
- [ ] Configuration data cached
- [ ] Query results cached when appropriate

**No Cache Pattern:**
```typescript
// BAD: Fetches config from DB on every request
async function getFeatureFlags() {
  return configRepository.find({ where: { type: 'feature_flag' } });
}
```

**Optimized Pattern:**
```typescript
// GOOD: Cached query (TypeORM)
async function getFeatureFlags() {
  return configRepository.find({
    where: { type: 'feature_flag' },
    cache: {
      id: 'feature_flags',
      milliseconds: 60000  // 1 minute cache
    }
  });
}

// OR: Application-level caching
const cache = new Map<string, { data: any; expires: number }>();

async function getFeatureFlags() {
  const cacheKey = 'feature_flags';
  const cached = cache.get(cacheKey);

  if (cached && cached.expires > Date.now()) {
    return cached.data;
  }

  const data = await configRepository.find({ where: { type: 'feature_flag' } });
  cache.set(cacheKey, { data, expires: Date.now() + 60000 });
  return data;
}
```

**Django Caching:**
```python
from django.core.cache import cache

def get_feature_flags():
    flags = cache.get('feature_flags')
    if flags is None:
        flags = list(Config.objects.filter(type='feature_flag'))
        cache.set('feature_flags', flags, 60)  # 60 seconds
    return flags
```

---

## 8. Batch Operations

### 8.1 Individual Inserts/Updates

**Search:**
```
Grep: \.save\(|\.create\(|\.update\(|\.insert\(|INSERT INTO|UPDATE.*SET
Glob: **/*.ts, **/*.js, **/*.py, **/*.java
```

**Check for inserts/updates in loops:**
- [ ] Bulk insert instead of loop
- [ ] Batch updates
- [ ] Upsert operations

**Slow Pattern:**
```typescript
// BAD: Individual inserts
async function createUsers(userData: UserData[]) {
  for (const data of userData) {
    await userRepository.save(new User(data));  // N queries
  }
}
```

**Optimized Pattern:**
```typescript
// GOOD: Bulk insert
async function createUsers(userData: UserData[]) {
  const users = userData.map(data => new User(data));
  await userRepository.save(users);  // Single query (or batched)
}

// OR: Using insert
async function createUsers(userData: UserData[]) {
  await userRepository.insert(userData);
}
```

**Prisma Batch:**
```typescript
// GOOD: createMany
await prisma.user.createMany({
  data: userData,
  skipDuplicates: true
});
```

**Django Bulk:**
```python
# GOOD: bulk_create
User.objects.bulk_create([
    User(**data) for data in user_data
], batch_size=1000)

# GOOD: bulk_update
User.objects.bulk_update(users, ['name', 'email'], batch_size=1000)
```

---

## 9. Raw Query Optimization

### 9.1 Complex Queries

**Search:**
```
Grep: \.query\(|\.raw\(|\$queryRaw|execute\(|raw_sql|RawSQL
Glob: **/*.ts, **/*.js, **/*.py
```

**Check raw queries for:**
- [ ] Using EXPLAIN to verify query plan
- [ ] Avoiding SELECT * in raw queries
- [ ] Proper parameterization

### 9.2 Count Optimization

**Search:**
```
Grep: \.count\(|COUNT\(\*\)|\.length|len\(
```

**Slow Pattern:**
```typescript
// BAD: Fetches all records to count
const users = await userRepository.find();
const count = users.length;
```

**Optimized Pattern:**
```typescript
// GOOD: Database-level count
const count = await userRepository.count();

// GOOD: Count with condition
const count = await userRepository.count({
  where: { status: 'active' }
});
```

---

## 10. Impact Classification

### Critical
- N+1 queries in list endpoints (confirmed)
- Missing indexes on frequently queried columns
- Unbounded queries on large tables
- Connection pool not configured

### High
- SELECT * fetching large columns
- Missing pagination on user-facing endpoints
- Long-running transactions with locks
- Individual inserts in loops (>10 items)

### Medium
- Over-eager loading of relations
- Missing query caching for static data
- Suboptimal transaction scope
- No connection validation

### Low
- Minor column selection improvements
- Optional index opportunities
- Query style preferences

---

## 11. Output Format

For each finding:

```
FINDING: [PERF03] Title
IMPACT: Critical|High|Medium|Low
EFFORT: Low|Medium|High
FILE: path/to/file.ts:lineNumber
DESCRIPTION: What the database issue is and its performance impact
SLOW_CODE:
```typescript
// Current implementation
```
OPTIMIZED_CODE:
```typescript
// Recommended optimized implementation
```
ESTIMATED_GAIN: "Reduces N+1 from 100 queries to 1, ~95% reduction in DB load"
BENCHMARK: "Measure with: EXPLAIN ANALYZE query; or ORM query logging"
```
