# Database Patterns Skill

Expert knowledge in database design, query optimization, migrations, and ORM patterns for modern applications.

## Core Competencies

- Schema design and normalization
- Query optimization and indexing strategies
- Migration management
- Connection pooling and performance tuning
- Repository pattern implementation
- Transaction management

## Patterns

### Repository Pattern with Prisma

```typescript
import { PrismaClient, User, Prisma } from '@prisma/client';

export interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findMany(params: FindManyUsersParams): Promise<PaginatedResult<User>>;
  create(data: CreateUserInput): Promise<User>;
  update(id: string, data: UpdateUserInput): Promise<User>;
  delete(id: string): Promise<void>;
}

export interface FindManyUsersParams {
  page?: number;
  limit?: number;
  search?: string;
  orderBy?: 'createdAt' | 'email' | 'name';
  order?: 'asc' | 'desc';
}

export interface PaginatedResult<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}

export interface CreateUserInput {
  email: string;
  name: string;
  password: string;
}

export interface UpdateUserInput {
  email?: string;
  name?: string;
  password?: string;
}

export class PrismaUserRepository implements UserRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } });
  }

  async findMany({
    page = 1,
    limit = 20,
    search,
    orderBy = 'createdAt',
    order = 'desc',
  }: FindManyUsersParams): Promise<PaginatedResult<User>> {
    const where: Prisma.UserWhereInput = search
      ? {
          OR: [
            { name: { contains: search, mode: 'insensitive' } },
            { email: { contains: search, mode: 'insensitive' } },
          ],
        }
      : {};

    const [data, total] = await this.prisma.$transaction([
      this.prisma.user.findMany({
        where,
        skip: (page - 1) * limit,
        take: limit,
        orderBy: { [orderBy]: order },
      }),
      this.prisma.user.count({ where }),
    ]);

    return {
      data,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    };
  }

  async create(data: CreateUserInput): Promise<User> {
    return this.prisma.user.create({ data });
  }

  async update(id: string, data: UpdateUserInput): Promise<User> {
    return this.prisma.user.update({ where: { id }, data });
  }

  async delete(id: string): Promise<void> {
    await this.prisma.user.delete({ where: { id } });
  }
}
```

### Transaction Wrapper

```typescript
export async function withTransaction<T>(
  prisma: PrismaClient,
  fn: (tx: Prisma.TransactionClient) => Promise<T>
): Promise<T> {
  return prisma.$transaction(fn, {
    maxWait: 5000,
    timeout: 10000,
    isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
  });
}
```

### Migration Best Practices

- Always write reversible migrations
- Add indexes for foreign keys and frequently queried columns
- Use `NOT NULL` with defaults when adding columns to existing tables
- Test migrations on a copy of production data before deploying

### Indexing Strategy

```sql
-- Composite index for common query patterns
CREATE INDEX idx_users_email_created ON users(email, created_at DESC);

-- Partial index for active records only
CREATE INDEX idx_orders_active ON orders(user_id, created_at)
  WHERE status != 'cancelled';

-- Full-text search index
CREATE INDEX idx_products_search ON products
  USING GIN(to_tsvector('english', name || ' ' || description));
```

## Connection Pooling

Use PgBouncer or built-in Prisma connection pooling. Set `connection_limit` based on:
- Number of application instances
- Database max connections
- Rule of thumb: `(max_db_connections / num_instances) - 2`

## Anti-patterns to Avoid

- N+1 queries — use `include` or `select` with relations
- Missing transactions for multi-step writes
- Unbounded queries without pagination
- Storing JSON blobs for frequently queried data
