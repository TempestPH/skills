---
name: prisma-database
description: >
  Especialista em banco de dados relacional com Prisma ORM v7 e PostgreSQL: performance, audit logs e soft delete.
  Use SEMPRE que o usuário pedir para criar, modificar ou planejar schemas Prisma, models, migrations, índices,
  audit logs, soft delete, enums, relações entre tabelas ou otimização de queries.
  Complemento da skill nextjs-fullstack, focando na camada de dados.
  Trigger: criar tabela, adicionar campo/coluna, schema.prisma, migration, @@index, @@map, @relation,
  foreign key, modelagem de dados, rastrear mudanças, log de auditoria, paginação cursor, N+1,
  ou qualquer operação com Prisma — mesmo sem mencionar "banco de dados" explicitamente.
  Se o usuário pede "criar uma entidade", "adicionar um model" ou "preciso de uma tabela para X", consulte esta skill.
---

# Prisma Database Expert — PostgreSQL

Referência para toda decisão de modelagem, schema Prisma v7, migrations, audit logs, soft delete e performance de banco de dados. Funciona como complemento da skill **nextjs-fullstack** — enquanto aquela cuida da camada de aplicação (services, actions, components), esta cuida exclusivamente da camada de dados.

---

## Prisma v7 — Setup Base

### Schema (`prisma/schema.prisma`)

```prisma
generator client {
  provider        = "prisma-client"
  output          = "../src/generated/prisma"
  previewFeatures = ["fullTextSearchPostgres"]
}

datasource db {
  provider = "postgresql"
}
```

### Configuração (`prisma/prisma.config.ts`)

```typescript
import path from "node:path"
import { defineConfig } from "prisma/config"

export default defineConfig({
  earlyAccess: true,
  schema: path.join(__dirname, "schema.prisma"),
})
```

As URLs de conexão (`DATABASE_URL`, `DIRECT_URL`, `SHADOW_DATABASE_URL`) são configuradas via variáveis de ambiente e referenciadas no `prisma.config.ts`, não mais no `schema.prisma` (mudança do v7).

### Instância do Client (`src/lib/db.ts`)

Seguir o padrão singleton da skill nextjs-fullstack, atualizado para v7:

```typescript
import { PrismaClient } from "@/generated/prisma"
import { softDeleteExtension } from "./prisma-extensions"

const globalForPrisma = globalThis as unknown as {
  prisma: ReturnType<typeof createPrismaClient> | undefined
}

function createPrismaClient() {
  return new PrismaClient({
    log:
      process.env.NODE_ENV === "development"
        ? ["query", "error", "warn"]
        : ["error"],
  }).$extends(softDeleteExtension)
}

export const db = globalForPrisma.prisma ?? createPrismaClient()

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = db
```

A diferença principal do v7: o import vem de `@/generated/prisma` (definido pelo `output` do generator) e não mais de `@prisma/client`.

---

## Seção 1 — Convenções de Schema

### 1.1 Nomenclatura

| Elemento | Convenção | Exemplo |
|---|---|---|
| Models | PascalCase singular | `User`, `Product`, `OrderItem` |
| Campos | camelCase | `firstName`, `createdAt`, `isActive` |
| Enums | PascalCase (nome) + SCREAMING_SNAKE_CASE (valores) | `enum Role { ADMIN USER }` |
| Tabelas (@@map) | snake_case plural | `@@map("order_items")` |
| Colunas (@map) | snake_case | `@map("first_name")` |
| Relações | camelCase, nome descritivo | `author`, `posts`, `orderItems` |

### 1.2 Campos Obrigatórios em Todo Model

Todo model deve incluir estes campos base — sem exceções:

```prisma
model Example {
  id        String    @id @default(cuid())
  createdAt DateTime  @default(now()) @map("created_at")
  updatedAt DateTime  @updatedAt @map("updated_at")
  deletedAt DateTime? @map("deleted_at")

  @@map("examples")
}
```

- **id**: `cuid()` para IDs string — melhor para sistemas distribuídos, não expõe sequência, e é URL-safe.
- **createdAt / updatedAt**: Rastreabilidade temporal obrigatória.
- **deletedAt**: Soft delete é o padrão. Nunca deletar fisicamente registros.
- **@@map**: Sempre mapear nomes de tabela para snake_case plural no PostgreSQL.

### 1.3 Relações

Definir sempre ambos os lados com `@relation` explícito:

```prisma
model User {
  id    String @id @default(cuid())
  posts Post[]

  @@map("users")
}

model Post {
  id       String @id @default(cuid())
  author   User   @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId String @map("author_id")

  @@map("posts")
}
```

Estratégias de `onDelete`:

| Estratégia | Quando usar | Exemplo |
|---|---|---|
| `Cascade` | Filho não faz sentido sem o pai | `Comment` → `Post` |
| `Restrict` | Filho é independente, bloquear deleção do pai | `Order` → `User` |
| `SetNull` | Relação opcional, campo pode ficar nulo | `Post.reviewerId` |

### 1.4 Enums

```prisma
enum Role {
  ADMIN
  USER
  MODERATOR
}
```

Usar enums para campos com valores fixos e finitos. Para valores que podem crescer dinamicamente, usar uma tabela separada.

### 1.5 Tipos PostgreSQL Otimizados

| Tipo de dado | Tipo Prisma | Atributo DB | Uso |
|---|---|---|---|
| Dinheiro/preço | `Decimal` | `@db.Decimal(10, 2)` | Valores financeiros (nunca Float) |
| Texto longo | `String` | `@db.Text` | Descrições, conteúdo |
| Texto curto com limite | `String` | `@db.VarChar(255)` | Nomes, emails |
| JSON flexível | `Json` | — | Metadata, configurações |
| Booleano com default | `Boolean` | `@default(true)` | Flags de estado |
| Timestamp com timezone | `DateTime` | `@db.Timestamptz` | Datas com fuso horário |

Para mais padrões avançados de schema, leia `references/schema-patterns.md`.

---

## Seção 2 — Soft Delete

### 2.1 Padrão no Schema

Todo model possui `deletedAt DateTime?`. Um registro é considerado "deletado" quando `deletedAt` não é `null`.

### 2.2 Extension de Soft Delete

```typescript
// src/lib/prisma-extensions.ts
import { Prisma } from "@/generated/prisma"

export const softDeleteExtension = Prisma.defineExtension({
  name: "softDelete",
  model: {
    $allModels: {
      async softDelete<T>(this: T, where: Record<string, unknown>) {
        const context = Prisma.getExtensionContext(this) as any
        return context.update({
          where,
          data: { deletedAt: new Date() },
        })
      },
      async restore<T>(this: T, where: Record<string, unknown>) {
        const context = Prisma.getExtensionContext(this) as any
        return context.update({
          where,
          data: { deletedAt: null },
        })
      },
    },
  },
  query: {
    $allModels: {
      async findMany({ args, query }) {
        if (args.where?.deletedAt === undefined) {
          args.where = { ...args.where, deletedAt: null }
        }
        return query(args)
      },
      async findFirst({ args, query }) {
        if (args.where?.deletedAt === undefined) {
          args.where = { ...args.where, deletedAt: null }
        }
        return query(args)
      },
      async findUnique({ args, query }) {
        const result = await query(args)
        if (
          result &&
          typeof result === "object" &&
          "deletedAt" in result &&
          result.deletedAt !== null
        ) {
          return null
        }
        return result
      },
      async count({ args, query }) {
        if (args.where?.deletedAt === undefined) {
          args.where = { ...args.where, deletedAt: null }
        }
        return query(args)
      },
    },
  },
})
```

### 2.3 Uso nos Services

```typescript
await db.user.softDelete({ id })   // Deletar (soft)
await db.user.restore({ id })       // Restaurar
await db.user.findMany()            // Listar (auto-filtra soft-deleted)
await db.user.findMany({ where: { deletedAt: { not: null } } })  // Só soft-deleted
await db.user.findMany({ where: { deletedAt: undefined } })      // Todos
```

---

## Seção 3 — Sistema de Audit Logs

Audit logs registram quem fez o quê, quando, e qual era o estado dos dados antes e depois da operação. São criados por entidade — apenas as entidades que o usuário define como auditáveis recebem um model de log.

### Regra fundamental

**Antes de criar ou modificar qualquer model no schema, perguntar ao usuário se a entidade deve ter audit logs.**

### 3.1 Enum compartilhado

```prisma
enum AuditAction {
  CREATE
  UPDATE
  DELETE
  RESTORE
}
```

### 3.2 Modelo de Log por Entidade

```prisma
model UserLog {
  id        String      @id @default(cuid())
  action    AuditAction
  entityId  String      @map("entity_id")
  userId    String?     @map("user_id")
  before    Json?
  after     Json?
  metadata  Json?
  createdAt DateTime    @default(now()) @map("created_at")

  user User? @relation("UserLogAuthor", fields: [userId], references: [id], onDelete: SetNull)

  @@index([entityId])
  @@index([userId])
  @@index([action])
  @@index([createdAt])
  @@map("user_logs")
}
```

### 3.3 Ao Modificar Models com Audit Logs

1. Campos adicionados aparecem automaticamente nos próximos logs (JSON é flexível)
2. Se relações do log forem removidas, atualizar o model de log
3. **Se a modificação adiciona nova funcionalidade em model sem audit logs, perguntar se deve adicionar agora**

Para implementação completa, leia `references/audit-log-system.md`.

---

## Seção 4 — Estratégia de Índices

### 4.1 O que indexar

- Campos usados frequentemente em `WHERE` e `ORDER BY`
- Foreign keys
- O par `deletedAt` + campos de filtro comuns (índice composto)

```prisma
@@index([deletedAt, isActive])
@@index([categoryId, deletedAt])
@@index([userId, deletedAt, createdAt])
```

### 4.2 Índices Parciais (Custom SQL)

```sql
-- prisma migrate dev --create-only
CREATE INDEX idx_products_active
  ON products (category_id, price)
  WHERE deleted_at IS NULL;
```

Para padrões avançados de índices, leia `references/schema-patterns.md`.

---

## Seção 5 — Performance

- **Select vs Include**: preferir `select` quando não precisa de todos os campos
- **Paginação cursor-based**: mais performático que offset para listas grandes
- **Bulk Operations**: `createMany`/`updateMany` ao invés de loops
- **Evitar N+1**: usar `include`/`select` com relações ao invés de queries em loop
- **Transactions**: `$transaction([...])` para batch, `$transaction(async tx => {})` para lógica condicional

---

## Seção 6 — Workflow de Migrations

1. Alterar `schema.prisma`
2. `npx prisma migrate dev --name descritivo_da_mudanca`
3. Revisar SQL gerado em `prisma/migrations/`
4. `npx prisma generate` (obrigatório no v7)
5. Testar

### Nomes de Migration

| Ação | Formato |
|---|---|
| Criar tabela | `create_[tabela]` |
| Adicionar campo | `add_[campo]_to_[tabela]` |
| Criar índice | `add_index_[tabela]_[campos]` |
| Adicionar audit log | `create_[entidade]_logs` |

### Checklist Pré-migration

- [ ] SQL gerado revisado?
- [ ] Risco de perda de dados?
- [ ] Índices adicionados para novos campos filtráveis?
- [ ] `deletedAt` presente no novo model?
- [ ] Model de audit log criado/atualizado?
- [ ] `prisma generate` executado?

---

## Fluxo de Decisão

### Ao Criar Novo Model
1. Definir campos com tipos PostgreSQL otimizados
2. Adicionar campos obrigatórios (`id`, `createdAt`, `updatedAt`, `deletedAt`)
3. Definir relações com `onDelete` strategy adequada
4. Adicionar `@@map` com nome snake_case plural
5. **Perguntar ao usuário: "Esta entidade deve ter audit logs?"**
6. Se sim → criar `[Entity]Log` + service de audit
7. Adicionar índices compostos com `deletedAt`
8. Criar migration com nome descritivo
9. Executar `npx prisma generate`

### Ao Modificar Model Existente
1. Avaliar impacto nos dados existentes
2. Verificar se existem audit logs para a entidade
3. **Se não tem audit logs, perguntar se deve adicionar agora**
4. Adicionar/atualizar índices se novos campos são filtráveis
5. Criar migration → revisar SQL → `npx prisma generate`

---

## Referências

- `references/schema-patterns.md` — Tipos PostgreSQL, composite unique, self-relations, JSON patterns, enum vs lookup table, many-to-many explícito
- `references/audit-log-system.md` — Service de audit log, integração com transactions, sanitização, queries de auditoria
