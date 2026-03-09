---
name: nextjs-fullstack
description: >
  Guia completo de desenvolvimento full-stack com Next.js (App Router), TypeScript, Tailwind CSS, Prisma, TanStack Query, Zod e NextAuth.js. Use esta skill sempre que o usuário pedir para criar, editar ou planejar qualquer parte do projeto: componentes, páginas, Server Actions, services, Route Handlers, schemas de validação, autenticação, configuração de banco de dados ou estrutura de pastas. Use também quando houver dúvidas sobre onde colocar código, como buscar dados, como estruturar mutations, como proteger rotas ou como organizar o projeto para facilitar futura extração de uma API separada.
---

# Next.js Full-Stack — Convenções e Padrões

Este documento é a fonte de verdade para decisões de arquitetura e geração de código no projeto. Toda alteração deve ser consistente com as regras aqui definidas.

---

## Stack Tecnológica

| Tecnologia | Papel |
|---|---|
| Next.js 14+ (App Router) | Framework full-stack |
| TypeScript | Tipagem estática (strict mode) |
| Tailwind CSS | Estilização |
| Prisma | ORM e acesso ao banco de dados |
| TanStack Query v5 | Estado de servidor no cliente |
| Zod | Validação de schemas (frontend e backend) |
| NextAuth.js v5 | Autenticação |

---

## Estrutura de Pastas

```
src/
├── app/                          # Rotas (App Router)
│   ├── (auth)/                   # Route group — rotas públicas de autenticação
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── register/
│   │       └── page.tsx
│   ├── (dashboard)/              # Route group — rotas protegidas
│   │   └── [feature]/
│   │       ├── page.tsx
│   │       └── [id]/
│   │           └── page.tsx
│   ├── api/
│   │   └── auth/
│   │       └── [...nextauth]/
│   │           └── route.ts      # Handler do NextAuth
│   ├── actions/                  # Server Actions globais (mutations)
│   ├── globals.css
│   └── layout.tsx
├── components/
│   ├── ui/                       # Componentes genéricos e reutilizáveis
│   └── [feature]/                # Componentes específicos de uma feature
├── hooks/                        # Custom hooks (useQuery, useMutation, etc.)
├── lib/
│   ├── auth.ts                   # Configuração do NextAuth
│   ├── db.ts                     # Instância singleton do Prisma
│   └── utils.ts                  # Funções utilitárias (cn, formatters, etc.)
├── providers/                    # React Context Providers
│   └── providers.tsx             # QueryClientProvider, SessionProvider, etc.
├── services/                     # DAL — Data Access Layer (organizado por entidade)
│   └── [entity].service.ts       # ex: user.service.ts, post.service.ts
├── types/                        # Tipos e interfaces TypeScript globais
│   └── index.ts
└── validations/                  # Schemas Zod (compartilhados entre client e server)
    └── [entity].validation.ts    # ex: user.validation.ts
```

**Regras de estrutura:**
- Cada nova feature ganha sua própria pasta em `components/[feature]/` e, se necessário, `app/actions/[feature]/`
- Componentes usados em apenas um lugar ficam colocalizados na pasta da página
- Nunca colocar lógica de banco de dados fora de `src/services/` ou `src/lib/db.ts`

---

## Seção 1 — Componentes e Páginas (Frontend)

### 1.1 Server vs. Client Components

**Regra principal:** todo componente é Server Component por padrão. Use `'use client'` apenas quando estritamente necessário.

| Usar Server Component | Usar `'use client'` |
|---|---|
| Buscar dados do banco | Interatividade (onClick, onChange) |
| Acessar variáveis de ambiente | Hooks de estado (useState, useEffect) |
| Operações sensíveis (sem expor ao browser) | TanStack Query (useQuery, useMutation) |
| Renderização de listas estáticas | APIs do browser (localStorage, window) |

**Padrão recomendado:** o Server Component busca os dados e os passa como props para o Client Component filho.

```tsx
// app/(dashboard)/users/page.tsx — Server Component
import { getUsers } from "@/services/user.service"
import { UserList } from "@/components/user/UserList"

export default async function UsersPage() {
  const users = await getUsers()
  return <UserList initialUsers={users} />
}
```

```tsx
// components/user/UserList.tsx — Client Component
"use client"

import { useQuery } from "@tanstack/react-query"

interface UserListProps {
  initialUsers: User[]
}

export function UserList({ initialUsers }: UserListProps) {
  const { data: users } = useQuery({
    queryKey: ["users"],
    queryFn: fetchUsers,
    initialData: initialUsers,
  })
  // ...
}
```

### 1.2 Convenções de Nomenclatura

| Item | Convenção | Exemplo |
|---|---|---|
| Componentes | PascalCase | `UserCard.tsx` |
| Hooks customizados | camelCase com prefixo `use` | `useUsers.ts` |
| Services e utils | camelCase | `user.service.ts` |
| Pastas de rotas | kebab-case | `/user-profile/` |
| Variáveis e funções | camelCase | `getUserById` |
| Constantes | SCREAMING_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Types e interfaces | PascalCase | `UserType`, `CreateUserInput` |
| Schemas Zod | camelCase com sufixo `Schema` | `createUserSchema` |

### 1.3 Tipagem TypeScript

- Usar `interface` para props de componentes e objetos que podem ser estendidos
- Usar `type` para unions, intersections e utilitários
- Nunca usar `any` — preferir `unknown` quando o tipo é incerto
- Sempre tipar o retorno de funções assíncronas explicitamente
- Usar path alias `@/` mapeado para `src/`

```typescript
// types/index.ts
export interface User {
  id: string
  name: string
  email: string
  createdAt: Date
}

export type CreateUserInput = Omit<User, "id" | "createdAt">
```

### 1.4 Estilização com Tailwind

- Todo estilo via classes Tailwind — sem `style={{ }}` inline
- Usar a função `cn()` de `@/lib/utils` para classes condicionais (combina `clsx` + `tailwind-merge`)
- Usar componentes de UI existentes no projeto; só criar novos se não houver equivalente
- Não duplicar classes — extrair para variáveis quando o bloco de classes for reutilizado

```typescript
// lib/utils.ts
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

---

## Seção 2 — Data Fetching com TanStack Query

### 2.1 Configuração do QueryClient

O `QueryClientProvider` vive em `src/providers/providers.tsx` e é wrapped no `layout.tsx` raiz.

```tsx
// src/providers/providers.tsx
"use client"

import { QueryClient, QueryClientProvider, isServer } from "@tanstack/react-query"

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // 1 minuto — evita refetch imediato no SSR
      },
    },
  })
}

let browserQueryClient: QueryClient | undefined = undefined

function getQueryClient() {
  if (isServer) return makeQueryClient()
  if (!browserQueryClient) browserQueryClient = makeQueryClient()
  return browserQueryClient
}

export function Providers({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient()
  return (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  )
}
```

### 2.2 Padrão de Prefetch com HydrationBoundary

Usar este padrão quando a página precisa de dados imediatos sem loading state visível ao usuário.

```tsx
// app/(dashboard)/posts/page.tsx
import { dehydrate, HydrationBoundary, QueryClient } from "@tanstack/react-query"
import { getPosts } from "@/services/post.service"
import { PostList } from "@/components/post/PostList"

export default async function PostsPage() {
  const queryClient = new QueryClient()

  await queryClient.prefetchQuery({
    queryKey: ["posts"],
    queryFn: getPosts,
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostList />
    </HydrationBoundary>
  )
}
```

### 2.3 Custom Hooks para Queries e Mutations

Centralizar todas as queries e mutations em `src/hooks/` para evitar duplicação.

```typescript
// hooks/useUsers.ts
"use client"

import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query"

export function useUsers() {
  return useQuery({
    queryKey: ["users"],
    queryFn: () => fetch("/api/users").then((r) => r.json()),
  })
}

export function useCreateUser() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (data: CreateUserInput) => {
      const result = await createUserAction(data)
      if (!result.success) throw new Error(result.error)
      return result.data
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["users"] })
    },
  })
}
```

---

## Seção 3 — Camada de Dados (DAL) e Prisma

> **Fronteira de extração futura:** toda a lógica desta seção é independente do Next.js. Ao extrair para uma API separada, os `services/` se tornam os controllers/use-cases — basta envolvê-los em rotas do novo framework.

### 3.1 Instância Singleton do Prisma

```typescript
// lib/db.ts
import { PrismaClient } from "@prisma/client"

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const db =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["query", "error", "warn"] : ["error"],
  })

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = db
```

### 3.2 Organização dos Services (por entidade)

Cada entidade do banco ganha seu próprio arquivo de service. O service é a única camada que importa `db` (Prisma).

```typescript
// services/user.service.ts
import { db } from "@/lib/db"
import type { User } from "@prisma/client"
import type { CreateUserInput } from "@/types"

export async function getUsers(): Promise<User[]> {
  return db.user.findMany({
    orderBy: { createdAt: "desc" },
  })
}

export async function getUserById(id: string): Promise<User | null> {
  return db.user.findUnique({ where: { id } })
}

export async function createUser(data: CreateUserInput): Promise<User> {
  return db.user.create({ data })
}

export async function updateUser(id: string, data: Partial<CreateUserInput>): Promise<User> {
  return db.user.update({ where: { id }, data })
}

export async function deleteUser(id: string): Promise<void> {
  await db.user.delete({ where: { id } })
}
```

**Regras dos services:**
- Nunca importar `db` fora de `src/services/` ou `src/lib/db.ts`
- Nunca importar services em Client Components (marcados com `'use client'`)
- Funções dos services devem ser pequenas, com responsabilidade única
- Erros do Prisma devem ser propagados (não silenciados) — o chamador trata

---

## Seção 4 — Validação com Zod

Schemas Zod ficam em `src/validations/` e são usados tanto em Server Actions quanto em Route Handlers.

```typescript
// validations/user.validation.ts
import { z } from "zod"

export const createUserSchema = z.object({
  name: z.string().min(2, "Nome deve ter ao menos 2 caracteres"),
  email: z.string().email("E-mail inválido"),
  password: z.string().min(8, "Senha deve ter ao menos 8 caracteres"),
})

export const updateUserSchema = createUserSchema.partial()

export type CreateUserInput = z.infer<typeof createUserSchema>
export type UpdateUserInput = z.infer<typeof updateUserSchema>
```

---

## Seção 5 — Server Actions (Mutations)

Server Actions ficam em `src/app/actions/` organizados por entidade. Elas são a interface entre o frontend e o DAL para operações de escrita.

### 5.1 Padrão de Server Action

```typescript
// app/actions/user.actions.ts
"use server"

import { revalidatePath } from "next/cache"
import { auth } from "@/lib/auth"
import { createUserSchema } from "@/validations/user.validation"
import { createUser, deleteUser } from "@/services/user.service"

export async function createUserAction(formData: unknown) {
  const session = await auth()
  if (!session) throw new Error("Não autorizado")

  const parsed = createUserSchema.safeParse(formData)
  if (!parsed.success) {
    return { success: false, error: parsed.error.flatten().fieldErrors }
  }

  try {
    const user = await createUser(parsed.data)
    revalidatePath("/users")
    return { success: true, data: user }
  } catch (error) {
    console.error("[createUserAction]", error)
    return { success: false, error: "Erro ao criar usuário" }
  }
}

export async function deleteUserAction(id: string) {
  const session = await auth()
  if (!session) throw new Error("Não autorizado")

  try {
    await deleteUser(id)
    revalidatePath("/users")
    return { success: true }
  } catch (error) {
    console.error("[deleteUserAction]", error)
    return { success: false, error: "Erro ao deletar usuário" }
  }
}
```

**Regras das Server Actions:**
- Sempre validar o input com Zod antes de qualquer operação
- Sempre verificar autenticação no início da action
- Sempre chamar `revalidatePath` ou `revalidateTag` após mutations bem-sucedidas
- Retornar objeto `{ success, data?, error? }` para o cliente poder reagir
- Nunca retornar dados sensíveis (senhas, tokens internos)
- Usar `try/catch` — nunca deixar erros do Prisma vazar sem tratamento

---

## Seção 6 — Autenticação com NextAuth.js

### 6.1 Configuração

```typescript
// lib/auth.ts
import NextAuth from "next-auth"
import { PrismaAdapter } from "@auth/prisma-adapter"
import { db } from "@/lib/db"
import Credentials from "next-auth/providers/credentials"

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(db),
  providers: [
    Credentials({
      credentials: {
        email: { label: "E-mail", type: "email" },
        password: { label: "Senha", type: "password" },
      },
      async authorize(credentials) {
        // Validar credenciais e buscar usuário no banco
        // Retornar null para negar acesso, objeto User para conceder
      },
    }),
  ],
  callbacks: {
    session({ session, token }) {
      if (token.sub) session.user.id = token.sub
      return session
    },
  },
  pages: {
    signIn: "/login",
  },
})
```

```typescript
// app/api/auth/[...nextauth]/route.ts
import { handlers } from "@/lib/auth"
export const { GET, POST } = handlers
```

### 6.2 Protegendo Rotas com Middleware

```typescript
// middleware.ts (raiz do projeto)
import { auth } from "@/lib/auth"

export default auth((req) => {
  const isLoggedIn = !!req.auth
  const isAuthRoute = req.nextUrl.pathname.startsWith("/login")

  if (!isLoggedIn && !isAuthRoute) {
    return Response.redirect(new URL("/login", req.nextUrl))
  }
})

export const config = {
  matcher: ["/((?!api|_next/static|_next/image|favicon.ico).*)"],
}
```

### 6.3 Acessando a Sessão

```typescript
// Em Server Components e Server Actions
import { auth } from "@/lib/auth"

const session = await auth()
if (!session?.user) redirect("/login")

// Em Client Components
import { useSession } from "next-auth/react"

const { data: session, status } = useSession()
```

---

## Seção 7 — Route Handlers (API Pública)

> **Nota:** no estágio atual do projeto, não há endpoints públicos. Esta seção documenta o padrão para quando forem necessários (ex: webhooks, app mobile, integrações externas). Os Route Handlers chamam os mesmos services da Seção 3 — garantindo que a lógica de negócio não fique duplicada.

### 7.1 Estrutura e Padrão

```
app/
└── api/
    ├── auth/
    │   └── [...nextauth]/
    │       └── route.ts      # Exclusivo do NextAuth
    └── [entity]/             # Adicionar quando necessário
        ├── route.ts          # GET (listar), POST (criar)
        └── [id]/
            └── route.ts      # GET (detalhe), PUT/PATCH, DELETE
```

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server"
import { auth } from "@/lib/auth"
import { getUsers, createUser } from "@/services/user.service"
import { createUserSchema } from "@/validations/user.validation"

export async function GET() {
  try {
    const session = await auth()
    if (!session) {
      return NextResponse.json({ error: "Não autorizado" }, { status: 401 })
    }

    const users = await getUsers()
    return NextResponse.json(users)
  } catch (error) {
    console.error("[GET /api/users]", error)
    return NextResponse.json({ error: "Erro interno" }, { status: 500 })
  }
}

export async function POST(request: NextRequest) {
  try {
    const session = await auth()
    if (!session) {
      return NextResponse.json({ error: "Não autorizado" }, { status: 401 })
    }

    const body = await request.json()
    const parsed = createUserSchema.safeParse(body)
    if (!parsed.success) {
      return NextResponse.json(
        { error: parsed.error.flatten().fieldErrors },
        { status: 422 }
      )
    }

    const user = await createUser(parsed.data)
    return NextResponse.json(user, { status: 201 })
  } catch (error) {
    console.error("[POST /api/users]", error)
    return NextResponse.json({ error: "Erro interno" }, { status: 500 })
  }
}
```

### 7.2 Middleware de Autenticação Reutilizável

```typescript
// lib/api-middleware.ts
import { NextRequest, NextResponse } from "next/server"
import { auth } from "@/lib/auth"

type Handler = (req: NextRequest, context?: unknown) => Promise<NextResponse>

export function withAuth(handler: Handler): Handler {
  return async (req, context) => {
    const session = await auth()
    if (!session) {
      return NextResponse.json({ error: "Não autorizado" }, { status: 401 })
    }
    return handler(req, context)
  }
}
```

**Regras dos Route Handlers:**
- Sempre verificar autenticação antes de processar
- Sempre validar body de requisições com Zod (status 422 para erros de validação)
- Sempre usar `try/catch` com log do erro e resposta padronizada
- Chamar os services do DAL — nunca acessar `db` diretamente no handler
- Usar `NextResponse.json()` para todas as respostas

---

## Seção 8 — Fluxo de Planejamento e Geração de Código

Ao receber uma solicitação de criação ou edição, siga esta sequência:

### Passo 1: Identificar o escopo
- É uma nova **página/rota**? → criar em `app/` + componente principal
- É um novo **componente**? → criar em `components/[feature]/` ou `components/ui/`
- É uma **mutation** (criar/editar/deletar)? → Server Action + validação Zod + service
- É uma **query de dados**? → service + custom hook TanStack Query

### Passo 2: Plano de implementação
Antes de gerar código, apresentar:
1. Arquivos que serão criados (com caminho completo)
2. Arquivos que serão modificados (com o que muda)
3. Decisão Server/Client Component e justificativa
4. Fluxo de dados (de onde vem, onde é exibido)

### Passo 3: Gerar o código
Seguir obrigatoriamente:
- Estrutura de pastas desta skill
- TypeScript strict (sem `any`)
- Validação Zod em qualquer input externo
- Autenticação verificada em Server Actions e Route Handlers
- `cn()` para classes Tailwind condicionais

### Passo 4: Checklist pós-geração
- [ ] Tipos exportados corretamente?
- [ ] Importações usando `@/` (sem paths relativos longos)?
- [ ] `'use client'` apenas onde necessário?
- [ ] Services não importados em Client Components?
- [ ] `revalidatePath`/`revalidateTag` chamados após mutations?
- [ ] Erros tratados com `try/catch` e mensagem amigável?

---

## Referência Rápida de Decisões

| Situação | Solução |
|---|---|
| Buscar dados para exibição | Server Component + service |
| Estado local do componente | `useState` em Client Component |
| Dados do servidor no cliente | TanStack Query `useQuery` + prefetch |
| Criar/editar/deletar dado | Server Action → service → `revalidatePath` |
| Validar formulário | Zod schema de `validations/` |
| Rota protegida | Middleware ou `auth()` no início da action/page |
| Endpoint para uso externo | Route Handler em `app/api/` → service |
| Variável de ambiente | `process.env.NOME_DA_VARIAVEL` |
| Classes Tailwind condicionais | `cn()` de `@/lib/utils` |
| Instância do banco | `db` de `@/lib/db` — somente em services |
