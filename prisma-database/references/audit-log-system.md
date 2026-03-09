# Sistema de Audit Logs — Implementação Completa

## Sumário

1. [Service de Audit Log](#service-de-audit-log)
2. [Integração com Services de Entidade](#integração-com-services-de-entidade)
3. [Helper de Serialização](#helper-de-serialização)
4. [Queries de Auditoria](#queries-de-auditoria)
5. [Exemplo Completo: Entidade Order](#exemplo-completo-entidade-order)

---

## Service de Audit Log

### Helper compartilhado

```typescript
// src/services/audit-log.helper.ts
import type { AuditAction } from "@/generated/prisma"

export interface AuditLogInput {
  action: AuditAction
  entityId: string
  userId?: string | null
  before?: Record<string, unknown> | null
  after?: Record<string, unknown> | null
  metadata?: Record<string, unknown> | null
}

export function sanitizeForLog(
  data: Record<string, unknown>
): Record<string, unknown> {
  const sensitiveFields = ["password", "passwordHash", "token", "secret", "refreshToken"]
  const sanitized = { ...data }

  for (const field of sensitiveFields) {
    if (field in sanitized) sanitized[field] = "[REDACTED]"
  }

  for (const [key, value] of Object.entries(sanitized)) {
    if (value instanceof Date) sanitized[key] = value.toISOString()
  }

  return sanitized
}
```

### Service específico por entidade

```typescript
// src/services/user-log.service.ts
import { db } from "@/lib/db"
import type { AuditLogInput } from "./audit-log.helper"
import { sanitizeForLog } from "./audit-log.helper"

export async function createUserLog(input: AuditLogInput) {
  return db.userLog.create({
    data: {
      action: input.action,
      entityId: input.entityId,
      userId: input.userId,
      before: input.before ? sanitizeForLog(input.before) : null,
      after: input.after ? sanitizeForLog(input.after) : null,
      metadata: input.metadata ?? null,
    },
  })
}

export async function getUserLogs(entityId: string) {
  return db.userLog.findMany({
    where: { entityId },
    orderBy: { createdAt: "desc" },
    include: {
      user: { select: { id: true, name: true, email: true } },
    },
  })
}
```

---

## Integração com Services de Entidade

O audit log é criado **dentro da mesma transaction** da operação principal — garantindo atomicidade.

### CREATE

```typescript
export async function createUser(
  data: CreateUserInput,
  performedBy?: string
): Promise<User> {
  return db.$transaction(async (tx) => {
    const user = await tx.user.create({ data })

    await tx.userLog.create({
      data: {
        action: "CREATE",
        entityId: user.id,
        userId: performedBy ?? null,
        before: null,
        after: sanitizeForLog(user as unknown as Record<string, unknown>),
      },
    })

    return user
  })
}
```

### UPDATE

```typescript
export async function updateUser(
  id: string,
  data: Partial<CreateUserInput>,
  performedBy?: string
): Promise<User> {
  return db.$transaction(async (tx) => {
    const before = await tx.user.findUniqueOrThrow({ where: { id } })
    const after = await tx.user.update({ where: { id }, data })

    await tx.userLog.create({
      data: {
        action: "UPDATE",
        entityId: id,
        userId: performedBy ?? null,
        before: sanitizeForLog(before as unknown as Record<string, unknown>),
        after: sanitizeForLog(after as unknown as Record<string, unknown>),
      },
    })

    return after
  })
}
```

### SOFT DELETE

```typescript
export async function deleteUser(
  id: string,
  performedBy?: string
): Promise<User> {
  return db.$transaction(async (tx) => {
    const before = await tx.user.findUniqueOrThrow({ where: { id } })

    const deleted = await tx.user.update({
      where: { id },
      data: { deletedAt: new Date() },
    })

    await tx.userLog.create({
      data: {
        action: "DELETE",
        entityId: id,
        userId: performedBy ?? null,
        before: sanitizeForLog(before as unknown as Record<string, unknown>),
        after: null,
        metadata: { softDelete: true },
      },
    })

    return deleted
  })
}
```

### RESTORE

```typescript
export async function restoreUser(
  id: string,
  performedBy?: string
): Promise<User> {
  return db.$transaction(async (tx) => {
    const restored = await tx.user.update({
      where: { id },
      data: { deletedAt: null },
    })

    await tx.userLog.create({
      data: {
        action: "RESTORE",
        entityId: id,
        userId: performedBy ?? null,
        before: null,
        after: sanitizeForLog(restored as unknown as Record<string, unknown>),
      },
    })

    return restored
  })
}
```

---

## Helper de Serialização

O `sanitizeForLog` garante:
1. Campos sensíveis (password, tokens) são redactados
2. Datas são convertidas para ISO string
3. O objeto é limpo para armazenamento seguro

Para adicionar campos sensíveis do projeto, estender a lista `sensitiveFields` em `audit-log.helper.ts`.

---

## Queries de Auditoria

### Timeline de uma entidade

```typescript
export async function getEntityTimeline(entityId: string) {
  return db.userLog.findMany({
    where: { entityId },
    orderBy: { createdAt: "desc" },
    select: {
      id: true,
      action: true,
      before: true,
      after: true,
      createdAt: true,
      user: { select: { name: true, email: true } },
    },
  })
}
```

### Diff entre before/after

```typescript
// src/lib/audit-utils.ts
export function computeDiff(
  before: Record<string, unknown> | null,
  after: Record<string, unknown> | null
): Array<{ field: string; from: unknown; to: unknown }> {
  if (!before || !after) return []

  return Object.keys(after)
    .filter((key) => JSON.stringify(before[key]) !== JSON.stringify(after[key]))
    .map((key) => ({ field: key, from: before[key], to: after[key] }))
}
```

---

## Exemplo Completo: Entidade Order

### Schema

```prisma
model Order {
  id         String      @id @default(cuid())
  userId     String      @map("user_id")
  status     OrderStatus @default(PENDING)
  total      Decimal     @db.Decimal(10, 2)
  notes      String?     @db.Text
  deletedAt  DateTime?   @map("deleted_at")
  createdAt  DateTime    @default(now()) @map("created_at")
  updatedAt  DateTime    @updatedAt @map("updated_at")

  user  User        @relation(fields: [userId], references: [id], onDelete: Restrict)
  items OrderItem[]
  logs  OrderLog[]

  @@index([userId, deletedAt])
  @@index([status, deletedAt])
  @@index([createdAt])
  @@map("orders")
}

model OrderLog {
  id        String      @id @default(cuid())
  action    AuditAction
  entityId  String      @map("entity_id")
  userId    String?     @map("user_id")
  before    Json?
  after     Json?
  metadata  Json?
  createdAt DateTime    @default(now()) @map("created_at")

  user  User?  @relation("OrderLogAuthor", fields: [userId], references: [id], onDelete: SetNull)
  order Order? @relation(fields: [entityId], references: [id], onDelete: Cascade)

  @@index([entityId])
  @@index([userId])
  @@index([action])
  @@index([createdAt])
  @@map("order_logs")
}
```

### Service

```typescript
// src/services/order.service.ts
import { db } from "@/lib/db"
import { sanitizeForLog } from "./audit-log.helper"
import type { OrderStatus } from "@/generated/prisma"

export async function updateOrderStatus(
  id: string,
  status: OrderStatus,
  performedBy?: string
) {
  return db.$transaction(async (tx) => {
    const before = await tx.order.findUniqueOrThrow({ where: { id } })
    const after = await tx.order.update({ where: { id }, data: { status } })

    await tx.orderLog.create({
      data: {
        action: "UPDATE",
        entityId: id,
        userId: performedBy ?? null,
        before: sanitizeForLog(before as unknown as Record<string, unknown>),
        after: sanitizeForLog(after as unknown as Record<string, unknown>),
        metadata: { changedField: "status", from: before.status, to: status },
      },
    })

    return after
  })
}
```

### Server Action

```typescript
// src/app/actions/order.actions.ts
"use server"

import { revalidatePath } from "next/cache"
import { auth } from "@/lib/auth"
import { createOrder } from "@/services/order.service"
import { createOrderSchema } from "@/validations/order.validation"

export async function createOrderAction(formData: unknown) {
  const session = await auth()
  if (!session?.user?.id) throw new Error("Não autorizado")

  const parsed = createOrderSchema.safeParse(formData)
  if (!parsed.success) {
    return { success: false, error: parsed.error.flatten().fieldErrors }
  }

  try {
    const order = await createOrder(parsed.data, session.user.id)
    revalidatePath("/orders")
    return { success: true, data: order }
  } catch (error) {
    console.error("[createOrderAction]", error)
    return { success: false, error: "Erro ao criar pedido" }
  }
}
```
