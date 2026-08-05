# Object-relational mapping (ORM)

**ORM** bridges object-oriented programming and relational databases: it maps tables to classes, rows to objects, and columns to attributes, so developers manipulate data through objects and method calls instead of writing raw SQL. The ORM framework handles translating between the two, manages relationships (one-to-many, many-to-many), and typically layers in extras like lazy loading, caching, and query generation.

## TL;DR
- ORMs map **tables → classes**, **rows → objects**, **columns → attributes**, and translate object operations into SQL under the hood.
- Big wins: less boilerplate, centralized schema definitions, built-in SQL-injection protection via parameterized queries, easier database portability.
- Biggest recurring pitfall: the **N+1 query problem** — fetching a list, then triggering one extra query per item to get related data. Fixed with eager loading.
- **Lazy loading** fetches related data only when accessed; **eager loading** fetches it upfront in the same/a joined query. Neither is universally correct — it depends on whether you'll actually use the related data.
- Drop to **raw SQL** when: the ORM generates an inefficient query, you need a complex aggregation/window function it can't express cleanly, or you're doing a bulk operation where ORM object overhead actually matters.

## What ORM does

### Key concepts
1. **Mapping**
   - Tables → classes (a `users` table maps to a `User` class).
   - Rows → objects (each row becomes a class instance).
   - Columns → attributes (columns become object properties/fields).
2. **Relationships**
   - **One-to-one** — a `User` has one `Profile`.
   - **One-to-many** — a `User` has many `Orders`.
   - **Many-to-many** — `Students` and `Courses`, typically via a junction table.
3. **Query abstraction** — method calls or fluent APIs (`User.query.get(id=20)`) replace hand-written SQL.
4. **Object lifecycle management** — the ORM tracks object state (new, modified, deleted) and syncs changes to the database.
5. **Metadata** — annotations, XML, or code-based config define how classes map to tables.

**SQL vs. ORM, side by side:**
```sql
SELECT id, name, email FROM users WHERE id = 20;
```
```python
user = User.query.get(id=20)
print(user.name, user.email)
```
The ORM translates the method call into the equivalent SQL, executes it, and maps the result back to a `User` object.

## A brief history

- **1990s** — early tools like TopLink (1996, Java) address the "impedance mismatch" between OOP and relational schemas.
- **2000s** — ORMs go mainstream: Hibernate (2001, Java) becomes the de facto Java standard; SQLAlchemy (2005, Python); Django ORM (2005, Python); Entity Framework (2008, .NET).
- **2010s** — micro-ORMs like Dapper (thin, performance-first) and modern ORMs like Prisma emerge, prioritizing either raw speed or developer experience.
- **2020s** — TypeScript-native ORMs (Prisma, TypeORM), GraphQL integration, and cloud-native/serverless-friendly features.

## How ORM works

1. **Define models** — classes representing tables, with attributes for columns and annotations/config for relationships.
   ```python
   from sqlalchemy import Column, Integer, String
   from sqlalchemy.ext.declarative import declarative_base

   Base = declarative_base()

   class User(Base):
       __tablename__ = 'users'
       id = Column(Integer, primary_key=True)
       name = Column(String)
       email = Column(String)
   ```
2. **Mapping metadata** — the ORM uses annotations, XML, or fluent config to map classes to tables (e.g., Hibernate's `@Entity`, `@Column`, `@OneToMany`).
3. **Query generation** — object-oriented queries translate to SQL:
   ```python
   User.query.filter_by(name="Alice").all()
   ```
   ```sql
   SELECT * FROM users WHERE name = 'Alice';
   ```
4. **Session / unit of work** — the ORM tracks a session/context of pending changes and batches them to the database.
   ```python
   user = User(name="Alice", email="alice@example.com")
   user.save()
   ```
5. **Relationship management** — foreign keys, lazy/eager loading, and cascading operations (deleting a parent can cascade-delete its children).
   ```java
   @Entity
   class User {
       @Id
       private Long id;
       private String name;
       @OneToMany(mappedBy = "user")
       private List<Order> orders;
   }
   ```
6. **Transaction management** — ORMs wrap operations in transactions to preserve atomicity/consistency (e.g., Entity Framework's `SaveChanges`).

## 🟢 Beginner: popular ORM frameworks

### Java
- **Hibernate** — the mature, de facto standard; supports JPA, HQL, inheritance, caching, lazy loading. Strong for enterprise/Spring apps.
  ```java
  @Entity
  public class User {
      @Id
      private Long id;
      private String name;
      public String getName() { return name; }
  }
  List<User> users = session.createQuery("FROM User WHERE name = :name", User.class)
      .setParameter("name", "Alice").getResultList();
  ```
- **EclipseLink** — open-source JPA implementation with XML/JSON support and multi-tenancy; strong Oracle integration.
- **Apache OpenJPA** — lightweight JPA implementation, good for small-to-medium projects.
- **jOOQ** — a DSL for type-safe SQL rather than a full ORM abstraction; generates code from your schema, gives fine-grained SQL control with compile-time safety.

### Python
- **SQLAlchemy** — flexible, with both a low-level Core layer and a higher-level ORM layer; highly customizable.
  ```python
  from sqlalchemy import create_engine, select
  from sqlalchemy.orm import sessionmaker

  engine = create_engine('sqlite:///example.db')
  Session = sessionmaker(bind=engine)
  session = Session()
  users = session.execute(select(User).filter_by(name="Alice")).scalars().all()
  ```
- **Django ORM** — tightly integrated with Django; migrations, admin interface, rapid development, beginner-friendly.
  ```python
  from django.db import models

  class User(models.Model):
      name = models.CharField(max_length=100)
      email = models.EmailField()

  users = User.objects.filter(name="Alice")
  ```
- **SQLObject**, **web2py DAL** — lighter-weight options for simple mappings or full-stack framework integration.

### PHP
- **Laravel Eloquent** — fluent, elegant syntax, tightly coupled to the Laravel ecosystem.
  ```php
  class User extends Model {
      protected $fillable = ['name', 'email'];
  }
  $users = User::where('name', 'Alice')->get();
  ```
- **Doctrine** — advanced, DQL-based, used heavily in Symfony/large PHP apps.
  ```php
  #[ORM\Entity]
  class User {
      #[ORM\Id]
      #[ORM\Column(type: 'integer')]
      private $id;
  }
  $users = $entityManager->getRepository(User::class)->findBy(['name' => 'Alice']);
  ```
- **CakePHP ORM**, **RedBeanPHP** — convention-over-configuration and zero-config options, respectively.

### .NET
- **Entity Framework (EF)** — Microsoft's flagship, deep .NET/LINQ integration, Code-First or Database-First.
  ```csharp
  public class AppDbContext : DbContext {
      public DbSet<User> Users { get; set; }
  }
  var users = context.Users.Where(u => u.Name == "Alice").ToList();
  ```
- **NHibernate** — mature, Hibernate-inspired, common in legacy .NET systems.
- **Dapper** — a micro-ORM prioritizing raw performance over abstraction.
  ```csharp
  var users = connection.Query<User>("SELECT * FROM Users WHERE Name = @Name", new { Name = "Alice" });
  ```

### JavaScript/TypeScript
- **Prisma** — modern, type-safe, schema-first, strong TypeScript integration.
  ```typescript
  const prisma = new PrismaClient();
  const users = await prisma.user.findMany({ where: { name: 'Alice' } });
  ```
- **TypeORM** — supports both Active Record and Data Mapper patterns, flexible cross-database support.
- **Sequelize** — mature, promise-based, widely used in Node.js.

### Comparison

| Framework | Language | Strengths | Weaknesses | Typical use |
|---|---|---|---|---|
| Hibernate | Java | Mature, scalable | Complex setup | Enterprise apps |
| SQLAlchemy | Python | Customizable, powerful | Steep learning curve | Web apps, data science |
| Django ORM | Python | Rapid development, simple | Framework-coupled | Web apps, CMS |
| Eloquent | PHP | Elegant, Laravel-integrated | Limited outside Laravel | PHP web apps |
| Doctrine | PHP | Flexible, enterprise-ready | Complex configuration | Symfony, large apps |
| Entity Framework | .NET | Deep .NET integration, scalable | Microsoft-centric | ASP.NET apps |
| Dapper | .NET | High performance, lightweight | Limited features (by design) | Performance-critical apps |
| Prisma | TypeScript | Developer-friendly, modern, type-safe | Newer, less mature | Full-stack JS apps |
| TypeORM | TypeScript | Flexible, TypeScript support | Inconsistent performance | Node.js apps |

## 🟡 Intermediate: lazy vs. eager loading, and the N+1 problem

### Lazy loading
Related data is fetched **only when accessed**, not as part of the initial query.
```python
user = User.query.get(1)
# no query yet for orders
print(user.orders)  # triggers a separate query here, the first time it's accessed
```
- **Pro**: avoids fetching data you might never use, keeps the initial query cheap.
- **Con**: if you access the relation for *many* objects in a loop, each access is a separate query — this is exactly how the N+1 problem happens.

### Eager loading
Related data is fetched **upfront**, in the same query (via `JOIN`) or in one additional batched query, regardless of whether you end up using it.
```python
users = User.query.options(joinedload(User.orders)).all()  # SQLAlchemy
# or, Django style:
users = User.objects.select_related('profile').prefetch_related('orders')
```
- **Pro**: one round-trip (or a small constant number) instead of one-per-object.
- **Con**: fetches data even when it turns out not to be needed, and can produce large result sets or expensive joins if overused on relations you don't actually need every time.

### The N+1 query problem

The single most common ORM performance pitfall. It happens when you fetch a list of N records, then trigger one additional query *per record* to fetch related data — N+1 queries total instead of 1 or 2.

```python
# BAD: N+1 queries
users = User.query.all()          # 1 query
for user in users:
    print(user.orders)            # N additional queries, one per user, via lazy loading
```
```python
# GOOD: eager loading collapses it to 1-2 queries
users = User.query.options(joinedload(User.orders)).all()  # 1 query (JOIN) or 2 (batched)
for user in users:
    print(user.orders)            # no additional queries — already loaded
```

Framework-specific fixes:
- Django: `select_related` (for `ForeignKey`/`OneToOne`, does a SQL `JOIN`) and `prefetch_related` (for `ManyToMany`/reverse FK, runs a separate batched query).
- SQLAlchemy: `joinedload()`, `selectinload()`, or `subqueryload()` depending on the shape of the relationship and result size.
- Laravel Eloquent: `User::with('orders')->get()` (eager loading via `with`).
- Prisma: `include: { orders: true }` on the query.

The N+1 problem is invisible in development with small datasets and a single test row — it becomes a real production incident only once N is a few hundred. This is exactly why query logging/profiling (Hibernate's `show_sql`, Django Debug Toolbar, Prisma's query logging) matters even for apps that "feel fine" locally.

### When to use raw SQL instead

ORMs are a productivity tool, not a mandate to never write SQL. Reach for raw SQL (or a query builder like jOOQ, or the ORM's own "escape hatch") when:

- **The ORM generates a demonstrably inefficient query** — an unnecessary subquery, a bad join order, or a query plan you can't tune through the ORM's API.
- **Complex aggregations or window functions** — `RANK() OVER (...)`, multi-level `GROUP BY` with `HAVING`, recursive CTEs — that are awkward or impossible to express cleanly through the ORM's query API.
- **Bulk operations** — inserting/updating thousands of rows where instantiating an ORM object per row adds real, measurable overhead. Most ORMs offer a bulk-insert escape hatch for exactly this reason (e.g., SQLAlchemy Core, Entity Framework's `BulkInsert` extensions).
- **Database-specific features** — full-text search syntax, JSON path queries, or vendor-specific functions the ORM doesn't model.
- **Performance-critical hot paths** — where every millisecond and every allocation is being measured, a micro-ORM (Dapper) or raw SQL avoids the ORM's mapping/tracking overhead entirely.

Most production codebases end up as a mix: ORM for the 90% of straightforward CRUD, raw SQL (often via the same ORM's "raw query" escape hatch, e.g. SQLAlchemy's `text()`) for the 10% that's genuinely complex or performance-sensitive. That's not a failure of the ORM — it's the intended design.

## 🟡 Intermediate: other advanced ORM features

- **Caching** — stores query results to skip re-hitting the database; e.g. Hibernate's second-level cache. For a shared cache layer instead of in-process ORM caching, see [`../caching/redis/redis.md`](../caching/redis/redis.md).
- **Migrations** — automate schema changes (new columns, new tables); e.g. Prisma's `migrate`, Django's `makemigrations`/`migrate`.
- **Transaction support** — wrap multiple operations atomically; e.g. Entity Framework's `TransactionScope`. See [`acid.md`](./acid.md) for what these guarantees actually mean underneath.
- **Connection pooling** — manage a pool of DB connections for concurrent load; e.g. SQLAlchemy's `pool_size`.
- **Event handling / lifecycle hooks** — run code on pre-save, post-delete, etc.; e.g. Doctrine's lifecycle callbacks.
- **Cross-database support** — abstract over MySQL/PostgreSQL/SQLite differences; e.g. Prisma's unified schema.
- **Type safety** — compile-time query/model checks; e.g. Prisma's generated TypeScript types.

## Advantages

| Benefit | Why it matters | Example |
|---|---|---|
| Productivity | Less boilerplate for CRUD | `User.save()` vs. hand-written `INSERT` |
| Maintainability | Models centralize schema (DRY) | Updating a model updates every query site |
| Abstraction | OOP syntax instead of raw SQL | `User.query.filter_by(name="Alice")` |
| Security | Parameterized queries by default | Prevents SQL injection out of the box |
| Portability | Abstracts the underlying DB vendor | Prisma's database-agnostic schema |
| Rapid development | Built-in migrations, scaffolding | Django's `makemigrations` |
| Relationship management | Simplifies complex joins/associations | Hibernate's `@ManyToMany` |

## Disadvantages

| Drawback | Why it happens | Mitigation |
|---|---|---|
| Performance overhead | Object↔SQL translation adds latency; N+1 is the classic case | Eager loading, query profiling |
| Abstraction leaks | Generated SQL can be suboptimal (bad joins, unnecessary subqueries) | Inspect generated SQL; drop to raw SQL when needed |
| Learning curve | Each ORM's API/quirks take real time to learn | Start with a simpler ORM or micro-ORM |
| Limited flexibility | Complex queries can be awkward to express | Raw SQL escape hatch |
| Debugging complexity | Generated queries can be hard to trace | Enable query logging |
| Resource usage | Session/object tracking costs memory | Tune session scope, avoid over-fetching |
| Vendor/framework lock-in | Some ORMs are tightly coupled to their framework | Prefer standalone ORMs if portability matters |

## Use cases

- **Web applications** — CRUD for users, content, e-commerce (Django for CMS, Laravel for APIs).
- **Enterprise systems** — complex relational models with transactions (Hibernate in Spring-based ERPs).
- **Rapid prototyping** — migrations + query builders accelerate MVPs (Prisma for startups).
- **Cross-database applications** — one codebase, multiple supported databases (Entity Framework with SQL Server and PostgreSQL).
- **Data-driven dashboards** — query/aggregate for analytics (SQLAlchemy in Flask dashboards).
- **Microservices** — per-service data access layers (TypeORM in Node.js microservices).

## Implementation examples

### SQLAlchemy (Python) — full CRUD with a relationship
```python
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship, sessionmaker

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)
    orders = relationship("Order", back_populates="user")

class Order(Base):
    __tablename__ = 'orders'
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id'))
    user = relationship("User", back_populates="orders")

engine = create_engine('sqlite:///example.db')
Base.metadata.create_all(engine)
Session = sessionmaker(bind=engine)
session = Session()

# Create
user = User(name="Alice", email="alice@example.com")
session.add(user)
session.commit()

# Read
users = session.query(User).filter_by(name="Alice").all()

# Update
user.name = "Bob"
session.commit()

# Delete
session.delete(user)
session.commit()
```

### Laravel Eloquent (PHP)
```php
class User extends Model {
    protected $fillable = ['name', 'email'];
    public function orders() {
        return $this->hasMany(Order::class);
    }
}

class Order extends Model {
    public function user() {
        return $this->belongsTo(User::class);
    }
}

// CRUD
$user = User::create(['name' => 'Alice', 'email' => 'alice@example.com']);
$users = User::where('name', 'Alice')->get();
$user->update(['name' => 'Bob']);
$user->delete();
```

### Prisma (TypeScript)
```typescript
// schema.prisma
model User {
  id     Int     @id @default(autoincrement())
  name   String
  email  String
  orders Order[]
}

model Order {
  id     Int  @id @default(autoincrement())
  userId Int
  user   User @relation(fields: [userId], references: [id])
}
```
```typescript
// index.ts
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

async function main() {
  const user = await prisma.user.create({
    data: { name: 'Alice', email: 'alice@example.com' },
  });
  const users = await prisma.user.findMany({ where: { name: 'Alice' } });
  await prisma.user.update({ where: { id: user.id }, data: { name: 'Bob' } });
  await prisma.user.delete({ where: { id: user.id } });
}
main().catch(console.error);
```

## 🔴 Advanced: performance optimization

1. **Indexing** — add indexes to frequently queried columns (Django's `index_together`, Prisma's `@index`).
2. **Batching** — batch insert/update operations to cut round-trips (Entity Framework's `BulkInsert` extensions).
3. **Caching** — layer in-memory or distributed caching (Redis) in front of frequent queries; see [`../caching/redis/redis.md`](../caching/redis/redis.md) for cache-aside/write-through patterns that pair naturally with an ORM's query layer.
4. **Selective queries** — avoid over-fetching columns you don't need (Prisma's `select` option).
5. **Connection pooling** — size pools for actual concurrent load (SQLAlchemy's `pool_size`).
6. **Lazy vs. eager, chosen deliberately** — pick per-relationship based on whether it's actually used on the hot path (Doctrine's `fetch="EAGER"`).

## Best practices

1. **Understand what the ORM generates** — inspect the actual SQL (SQLAlchemy's `explain`, Hibernate's `show_sql`) before assuming a query is efficient.
2. **Avoid N+1 by default on any list + relation access** — reach for eager loading (`select_related`, `with`, `include`) as the default, not an afterthought.
3. **Define precise column types and indexes** — don't leave this to defaults.
4. **Wrap multi-step operations in transactions** — see [`acid.md`](./acid.md) for what atomicity actually buys you here.
5. **Rely on the ORM's parameterized queries** for SQL-injection protection — don't hand-build query strings.
6. **Profile and log queries** in staging/production, not just development with tiny datasets.
7. **Use the ORM's migration tooling** rather than hand-editing schema out of band.
8. **Combine with raw SQL deliberately** for the cases outlined above — this is normal, not a workaround.
9. **Pick the ORM for the project's actual needs** — Dapper for raw performance, Prisma for TypeScript-heavy teams, Hibernate for large Java/Spring systems.

## Further reading
- [Martin Fowler: ORM Hate (and a defense)](https://martinfowler.com/bliki/OrmHate.html)
- [SQLAlchemy documentation](https://docs.sqlalchemy.org/)
- [Prisma documentation](https://www.prisma.io/docs)
- [Django ORM optimization guide](https://docs.djangoproject.com/en/stable/topics/db/optimization/)
