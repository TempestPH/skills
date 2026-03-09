# Padrões Avançados de Schema — PostgreSQL + Prisma v7

## Sumário

1. [Tipos PostgreSQL Otimizados](#tipos-postgresql-otimizados)
2. [Composite Unique Constraints](#composite-unique-constraints)
3. [Self-Relations (Hierarquias)](#self-relations-hierarquias)
4. [JSON Patterns](#json-patterns)
5. [Enum vs Lookup Table](#enum-vs-lookup-table)
6. [Índices Avançados](#índices-avançados)
7. [Padrão de Timestamps](#padrão-de-timestamps)
8. [Many-to-Many Explícito](#many-to-many-explícito)

---

## Tipos PostgreSQL Otimizados

### Valores financeiros — nunca usar Float

```prisma
model Product {
  price    Decimal @db.Decimal(10, 2)
  discount Decimal @db.Decimal(5, 2) @default(0)
  tax      Decimal @db.Decimal(5, 4)
}
```

`Float` tem problemas de precisão binária (ex: 0.1 + 0.2 ≠ 0.3). `Decimal` armazena o valor exato.

### Textos

```prisma
model Article {
  title   String  @db.VarChar(255)
  slug    String  @unique @db.VarChar(300)
  summary String? @db.VarChar(500)
  content String  @db.Text
}
```

### Timestamps com timezone

```prisma
model Event {
  startsAt DateTime @db.Timestamptz
  endsAt   DateTime @db.Timestamptz
}
```

### Arrays nativos

```prisma
model Post {
  tags String[] @default([])
}
```

---

## Composite Unique Constraints

```prisma
model TeamMember {
  id     String @id @default(cuid())
  teamId String @map("team_id")
  userId String @map("user_id")
  role   String

  team Team @relation(fields: [teamId], references: [id], onDelete: Cascade)
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([teamId, userId])
  @@map("team_members")
}
```

Cenários comuns:
- `@@unique([tenantId, email])` — email único por tenant
- `@@unique([orderId, productId])` — produto único por pedido
- `@@unique([userId, slug])` — slug único por usuário

---

## Self-Relations (Hierarquias)

```prisma
model Category {
  id       String  @id @default(cuid())
  name     String
  parentId String? @map("parent_id")

  parent   Category?  @relation("CategoryTree", fields: [parentId], references: [id], onDelete: SetNull)
  children Category[] @relation("CategoryTree")

  deletedAt DateTime? @map("deleted_at")
  createdAt DateTime  @default(now()) @map("created_at")
  updatedAt DateTime  @updatedAt @map("updated_at")

  @@index([parentId])
  @@map("categories")
}
```

Para hierarquias profundas (> 3 níveis), materializar o path:

```prisma
model Category {
  path  String @default("")  // "root/parent/child"
  depth Int    @default(0)

  @@index([path])
}
```

---

## JSON Patterns

```prisma
model User {
  metadata Json? @default("{}")
}
```

| Usar JSON | Usar campos tipados |
|---|---|
| Dados de terceiros (webhooks) | Dados usados em filtros frequentes |
| Configurações que variam por usuário | Dados com validação de tipo rigorosa |
| Dados que mudam de estrutura | Dados usados em JOINs |

```typescript
const users = await db.user.findMany({
  where: {
    metadata: {
      path: ["preferences", "theme"],
      equals: "dark",
    },
  },
})
```

---

## Enum vs Lookup Table

**Usar Enum** quando: valores fixos, menos de ~10 valores, sem metadata extra.

**Usar Lookup Table** quando: valores dinâmicos, precisam de metadata (nome traduzido, ícone, cor), usuário pode criar novos.

```prisma
model Tag {
  id    String @id @default(cuid())
  name  String @unique
  slug  String @unique
  color String @default("#000000")

  posts PostTag[]

  @@map("tags")
}
```

---

## Índices Avançados

### Índice para soft delete + filtro

```prisma
@@index([deletedAt, status])
@@index([deletedAt, categoryId])
@@index([deletedAt, createdAt])
```

### Índice parcial via Custom SQL

```sql
CREATE INDEX idx_orders_active_status
  ON orders (status, created_at)
  WHERE deleted_at IS NULL;
```

### Índice GIN para arrays

```prisma
model Post {
  tags String[]
  @@index([tags], type: Gin)
}
```

### Índice GIN para full-text search

```sql
CREATE INDEX idx_products_search
  ON products
  USING gin(to_tsvector('portuguese', name || ' ' || coalesce(description, '')));
```

---

## Padrão de Timestamps

```prisma
model Order {
  confirmedAt DateTime? @map("confirmed_at") @db.Timestamptz
  shippedAt   DateTime? @map("shipped_at") @db.Timestamptz
  deliveredAt DateTime? @map("delivered_at") @db.Timestamptz
  cancelledAt DateTime? @map("cancelled_at") @db.Timestamptz
}
```

Timestamps de transição de estado são mais confiáveis e indexáveis que depender apenas de audit logs.

---

## Many-to-Many Explícito

```prisma
model PostTag {
  id        String   @id @default(cuid())
  postId    String   @map("post_id")
  tagId     String   @map("tag_id")
  createdAt DateTime @default(now()) @map("created_at")

  post Post @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag  Tag  @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@unique([postId, tagId])
  @@index([tagId])
  @@map("post_tags")
}
```

Vantagens: campos extras, controle de `onDelete`, indexação precisa, soft delete na relação.
