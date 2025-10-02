# Plano de Integração RAG com Upstash Vector Database

## Objetivo
Implementar um sistema de Retrieval-Augmented Generation (RAG) usando Upstash Vector Database para fornecer contexto relevante às conversas de chat baseadas em uma base de conhecimento gerenciada pelo admin.

## Arquitetura Proposta

### 1. Componentes Principais

#### 1.1 Vector Database (Upstash)
- **Provider**: Upstash Vector Database
- **Integração**: Via AI SDK (Vercel) com embeddings OpenRouter
- **Embedding Model**: text-embedding-3-small via OpenRouter
- **Dimensões**: 1536 (padrão OpenAI)

#### 1.2 Upload de Documentos (Admin)
- Interface no painel admin para upload de arquivos
- **Formatos suportados**: TXT, MD (Markdown)
- Processamento e chunking de documentos
- Armazenamento de metadados (nome, tipo, data, tags, status)
- **Multi-tenant**: Isolamento por userId e/ou workspaceId

#### 1.3 Injeção de Contexto (Chat)
- Busca semântica no vector database durante conversas
- Recuperação dos chunks mais relevantes (top-k)
- Injeção no prompt do sistema antes da resposta do LLM
- Filtragem por tenant (userId/workspaceId) e status

---

## Implementação Técnica

### 2. Setup Inicial

#### 2.1 Dependências NPM
```bash
npm install @upstash/vector
# AI SDK e OpenRouter já estão instalados no projeto
```

#### 2.2 Variáveis de Ambiente
Adicionar ao `.env.local` e `.env.example`:
```env
# Upstash Vector
UPSTASH_VECTOR_REST_URL=https://your-vector-db.upstash.io
UPSTASH_VECTOR_REST_TOKEN=your_token_here

# Configurações RAG
RAG_TOP_K=5 # Número de chunks a recuperar
RAG_CHUNK_SIZE=800 # Tamanho dos chunks em tokens (500-800)
RAG_CHUNK_OVERLAP=100 # Overlap entre chunks
RAG_MAX_CONTEXT_TOKENS=2000 # Limite de tokens no contexto injetado
```

#### 2.3 Schema Prisma - Novos Modelos
Baseado no prompt `upstash-knowledge-base.md`:

```prisma
model KnowledgeBaseEntry {
  id          String   @id @default(cuid())
  title       String
  content     String   @db.Text
  tags        String[] // Array de tags para categorização
  status      EntryStatus @default(ACTIVE)

  // Multi-tenant: pelo menos um deve estar preenchido
  userId      String?
  workspaceId String?

  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  chunks      KnowledgeChunk[]

  @@index([userId])
  @@index([workspaceId])
  @@index([status])
  @@map("knowledge_base_entries")
}

model KnowledgeChunk {
  id        String   @id @default(cuid())
  entryId   String
  ordinal   Int      // Ordem do chunk no documento
  content   String   @db.Text
  tokens    Int?     // Quantidade de tokens no chunk
  vectorId  String   // ID do vetor no Upstash (formato: entryId:ordinal)

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  entry     KnowledgeBaseEntry @relation(fields: [entryId], references: [id], onDelete: Cascade)

  @@unique([entryId, ordinal])
  @@index([entryId])
  @@map("knowledge_chunks")
}

enum EntryStatus {
  ACTIVE
  DRAFT
  ARCHIVED
}
```

---

### 3. Backend - API Routes

#### 3.1 CRUD Knowledge Base Entries
**Rotas**:
- `POST /api/admin/knowledge` - Criar entrada e indexar
- `PUT /api/admin/knowledge/:id` - Atualizar e reindexar
- `DELETE /api/admin/knowledge/:id` - Deletar entrada e vetores
- `POST /api/admin/knowledge/:id/reindex` - Reindexar manualmente
- `GET /api/admin/knowledge` - Listar com paginação/filtros

**Exemplo - Criar Entrada**:
```typescript
// src/app/api/admin/knowledge/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { auth } from '@clerk/nextjs/server';
import { z } from 'zod';
import { prisma } from '@/lib/db';
import { isAdmin, getUserFromClerkId } from '@/lib/auth-utils';
import { indexEntry } from '@/lib/knowledge/indexer';

const CreateEntrySchema = z.object({
  title: z.string().min(1).max(500),
  content: z.string().min(1).max(50000),
  tags: z.array(z.string()).optional().default([]),
  status: z.enum(['ACTIVE', 'DRAFT', 'ARCHIVED']).optional().default('ACTIVE'),
  workspaceId: z.string().optional(),
});

export async function POST(req: NextRequest) {
  const { userId } = await auth();
  if (!userId || !(await isAdmin(userId))) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const dbUser = await getUserFromClerkId(userId);
  const body = await req.json();
  const parsed = CreateEntrySchema.safeParse(body);

  if (!parsed.success) {
    return NextResponse.json(
      { error: 'Dados inválidos', issues: parsed.error.flatten() },
      { status: 400 }
    );
  }

  const { title, content, tags, status, workspaceId } = parsed.data;

  // Criar entrada no banco
  const entry = await prisma.knowledgeBaseEntry.create({
    data: {
      title,
      content,
      tags,
      status,
      userId: dbUser.id,
      workspaceId,
    },
  });

  // Indexar assincronamente (ou sincronamente para MVP)
  try {
    await indexEntry(entry);
  } catch (error) {
    // Marcar como falha mas retornar entrada criada
    console.error('Erro ao indexar entrada:', error);
  }

  return NextResponse.json(entry, { status: 201 });
}

export async function GET(req: NextRequest) {
  const { userId } = await auth();
  if (!userId || !(await isAdmin(userId))) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { searchParams } = new URL(req.url);
  const page = parseInt(searchParams.get('page') || '1');
  const limit = parseInt(searchParams.get('limit') || '20');
  const status = searchParams.get('status') as 'ACTIVE' | 'DRAFT' | 'ARCHIVED' | null;
  const search = searchParams.get('search');

  const where: any = {};
  if (status) where.status = status;
  if (search) {
    where.OR = [
      { title: { contains: search, mode: 'insensitive' } },
      { tags: { has: search } },
    ];
  }

  const [entries, total] = await Promise.all([
    prisma.knowledgeBaseEntry.findMany({
      where,
      include: { chunks: true },
      orderBy: { updatedAt: 'desc' },
      skip: (page - 1) * limit,
      take: limit,
    }),
    prisma.knowledgeBaseEntry.count({ where }),
  ]);

  return NextResponse.json({
    entries,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit),
    },
  });
}
```

**Exemplo - Update e Delete**:
```typescript
// src/app/api/admin/knowledge/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { auth } from '@clerk/nextjs/server';
import { prisma } from '@/lib/db';
import { isAdmin } from '@/lib/auth-utils';
import { indexEntry, deleteEntryVectors } from '@/lib/knowledge/indexer';

export async function PUT(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const { userId } = await auth();
  if (!userId || !(await isAdmin(userId))) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { id } = params;
  const body = await req.json();

  // Atualizar entrada
  const entry = await prisma.knowledgeBaseEntry.update({
    where: { id },
    data: {
      title: body.title,
      content: body.content,
      tags: body.tags,
      status: body.status,
    },
  });

  // Deletar chunks e vetores antigos
  await deleteEntryVectors(id);

  // Reindexar
  await indexEntry(entry);

  return NextResponse.json(entry);
}

export async function DELETE(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const { userId } = await auth();
  if (!userId || !(await isAdmin(userId))) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { id } = params;

  // Deletar vetores do Upstash
  await deleteEntryVectors(id);

  // Deletar entrada (cascade deleta chunks)
  await prisma.knowledgeBaseEntry.delete({
    where: { id },
  });

  return NextResponse.json({ success: true });
}
```

**Exemplo - Reindex**:
```typescript
// src/app/api/admin/knowledge/[id]/reindex/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { auth } from '@clerk/nextjs/server';
import { prisma } from '@/lib/db';
import { isAdmin } from '@/lib/auth-utils';
import { indexEntry, deleteEntryVectors } from '@/lib/knowledge/indexer';

export async function POST(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const { userId } = await auth();
  if (!userId || !(await isAdmin(userId))) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { id } = params;

  const entry = await prisma.knowledgeBaseEntry.findUnique({
    where: { id },
  });

  if (!entry) {
    return NextResponse.json({ error: 'Entrada não encontrada' }, { status: 404 });
  }

  // Deletar chunks e vetores antigos
  await deleteEntryVectors(id);

  // Reindexar
  await indexEntry(entry);

  return NextResponse.json({ success: true, message: 'Reindexação concluída' });
}
```

---

### 4. Biblioteca RAG Core

#### 4.1 Vector Client Setup
```typescript
// src/lib/knowledge/vector-client.ts
import { Index } from '@upstash/vector';

if (!process.env.UPSTASH_VECTOR_REST_URL || !process.env.UPSTASH_VECTOR_REST_TOKEN) {
  throw new Error('Upstash Vector credentials not configured');
}

export const vectorIndex = new Index({
  url: process.env.UPSTASH_VECTOR_REST_URL,
  token: process.env.UPSTASH_VECTOR_REST_TOKEN,
});
```

#### 4.2 Embeddings Generator (OpenRouter)
```typescript
// src/lib/knowledge/embeddings.ts
import { embed, embedMany } from 'ai';
import { createOpenAI } from '@ai-sdk/openai';

const openrouter = createOpenAI({
  apiKey: process.env.OPENROUTER_API_KEY,
  baseURL: 'https://openrouter.ai/api/v1',
});

/**
 * Gera embedding para um único texto usando OpenRouter
 */
export async function generateEmbedding(text: string): Promise<number[]> {
  const { embedding } = await embed({
    model: openrouter.embedding('openai/text-embedding-3-small'),
    value: text,
  });

  return embedding;
}

/**
 * Gera embeddings para múltiplos textos em batch
 */
export async function generateEmbeddings(texts: string[]): Promise<number[][]> {
  const { embeddings } = await embedMany({
    model: openrouter.embedding('openai/text-embedding-3-small'),
    values: texts,
  });

  return embeddings;
}
```

#### 4.3 Text Chunking
```typescript
// src/lib/knowledge/chunking.ts

export interface ChunkOptions {
  chunkSize: number; // Em caracteres
  overlap: number; // Em caracteres
}

export interface Chunk {
  content: string;
  ordinal: number;
}

/**
 * Divide texto em chunks com overlap
 * Baseado em caracteres (aproximação de tokens)
 */
export function chunkText(
  text: string,
  options?: Partial<ChunkOptions>
): Chunk[] {
  const chunkSize = options?.chunkSize || parseInt(process.env.RAG_CHUNK_SIZE || '800');
  const overlap = options?.overlap || parseInt(process.env.RAG_CHUNK_OVERLAP || '100');

  const chunks: Chunk[] = [];
  let startIndex = 0;
  let ordinal = 0;

  // Normalizar espaços em branco
  const normalizedText = text.replace(/\s+/g, ' ').trim();

  while (startIndex < normalizedText.length) {
    const endIndex = Math.min(startIndex + chunkSize, normalizedText.length);
    let chunk = normalizedText.slice(startIndex, endIndex);

    // Tentar quebrar em limite de sentença se possível
    if (endIndex < normalizedText.length) {
      const lastPeriod = chunk.lastIndexOf('. ');
      const lastNewline = chunk.lastIndexOf('\n');
      const breakPoint = Math.max(lastPeriod, lastNewline);

      if (breakPoint > chunkSize * 0.5) {
        chunk = chunk.slice(0, breakPoint + 1);
      }
    }

    chunk = chunk.trim();
    if (chunk.length > 0) {
      chunks.push({ content: chunk, ordinal });
      ordinal++;
    }

    // Avançar com overlap
    startIndex += chunk.length - overlap;
    if (startIndex <= 0) startIndex = chunk.length; // Evitar loop infinito
  }

  return chunks;
}

/**
 * Estima quantidade de tokens (aproximação: 1 token ≈ 4 caracteres)
 */
export function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);
}
```

#### 4.4 Document Indexer
```typescript
// src/lib/knowledge/indexer.ts
import { prisma } from '@/lib/db';
import { vectorIndex } from './vector-client';
import { chunkText, estimateTokens } from './chunking';
import { generateEmbeddings } from './embeddings';
import type { KnowledgeBaseEntry } from '@prisma/client';

/**
 * Indexa uma entrada na base de conhecimento
 * 1. Faz chunking do conteúdo
 * 2. Gera embeddings
 * 3. Upsert no Upstash Vector
 * 4. Salva chunks no banco de dados
 */
export async function indexEntry(entry: KnowledgeBaseEntry): Promise<void> {
  // 1. Chunking
  const chunks = chunkText(entry.content);

  if (chunks.length === 0) {
    throw new Error('Nenhum chunk gerado para a entrada');
  }

  // 2. Gerar embeddings em batch
  const embeddings = await generateEmbeddings(chunks.map((c) => c.content));

  // 3. Preparar vetores para Upstash
  const vectors = chunks.map((chunk, index) => ({
    id: `${entry.id}:${chunk.ordinal}`,
    vector: embeddings[index],
    metadata: {
      entryId: entry.id,
      ordinal: chunk.ordinal,
      userId: entry.userId,
      workspaceId: entry.workspaceId,
      status: entry.status,
      title: entry.title,
      tags: entry.tags,
    },
  }));

  // 4. Upsert no Upstash Vector
  await vectorIndex.upsert(vectors);

  // 5. Salvar chunks no banco de dados
  await prisma.knowledgeChunk.createMany({
    data: chunks.map((chunk, index) => ({
      entryId: entry.id,
      ordinal: chunk.ordinal,
      content: chunk.content,
      tokens: estimateTokens(chunk.content),
      vectorId: `${entry.id}:${chunk.ordinal}`,
    })),
  });

  console.log(`✅ Entrada ${entry.id} indexada com ${chunks.length} chunks`);
}

/**
 * Deleta todos os vetores e chunks de uma entrada
 */
export async function deleteEntryVectors(entryId: string): Promise<void> {
  // 1. Buscar todos os chunks
  const chunks = await prisma.knowledgeChunk.findMany({
    where: { entryId },
    select: { vectorId: true },
  });

  // 2. Deletar vetores do Upstash
  if (chunks.length > 0) {
    const vectorIds = chunks.map((c) => c.vectorId);
    await vectorIndex.delete(vectorIds);
  }

  // 3. Deletar chunks do banco (cascade já faria isso, mas explicitamos)
  await prisma.knowledgeChunk.deleteMany({
    where: { entryId },
  });

  console.log(`🗑️ Deletados ${chunks.length} vetores da entrada ${entryId}`);
}
```

#### 4.5 Vector Search (RAG Retrieval)
```typescript
// src/lib/knowledge/search.ts
import { vectorIndex } from './vector-client';
import { generateEmbedding } from './embeddings';

export interface SearchOptions {
  userId?: string;
  workspaceId?: string;
  topK?: number;
  minScore?: number;
}

export interface SearchResult {
  id: string;
  score: number;
  content: string;
  entryId: string;
  title: string;
  tags: string[];
  metadata: Record<string, any>;
}

/**
 * Busca chunks relevantes na base de conhecimento
 */
export async function searchKnowledgeBase(
  query: string,
  options: SearchOptions = {}
): Promise<SearchResult[]> {
  const {
    userId,
    workspaceId,
    topK = parseInt(process.env.RAG_TOP_K || '5'),
    minScore = 0.7,
  } = options;

  // 1. Gerar embedding da query
  const queryEmbedding = await generateEmbedding(query);

  // 2. Preparar filtros de tenant
  const filter: Record<string, any> = { status: 'ACTIVE' };
  if (userId) filter.userId = userId;
  if (workspaceId) filter.workspaceId = workspaceId;

  // 3. Buscar vetores similares no Upstash
  const results = await vectorIndex.query({
    vector: queryEmbedding,
    topK,
    includeMetadata: true,
    filter,
  });

  // 4. Filtrar por score mínimo e formatar resultados
  const formattedResults: SearchResult[] = results
    .filter((r) => r.score >= minScore)
    .map((result) => ({
      id: result.id,
      score: result.score,
      content: result.metadata?.content as string || '',
      entryId: result.metadata?.entryId as string,
      title: result.metadata?.title as string || '',
      tags: (result.metadata?.tags as string[]) || [],
      metadata: result.metadata || {},
    }));

  return formattedResults;
}

/**
 * Formata resultados de busca em texto de contexto
 */
export function formatContextFromResults(results: SearchResult[]): string {
  if (results.length === 0) return '';

  const maxTokens = parseInt(process.env.RAG_MAX_CONTEXT_TOKENS || '2000');
  let totalTokens = 0;
  const contextParts: string[] = [];

  for (const result of results) {
    const chunkTokens = Math.ceil(result.content.length / 4); // Aproximação
    if (totalTokens + chunkTokens > maxTokens) break;

    contextParts.push(
      `[Fonte: ${result.title} | Relevância: ${(result.score * 100).toFixed(1)}%]\n${result.content}`
    );
    totalTokens += chunkTokens;
  }

  return contextParts.join('\n\n---\n\n');
}
```

---

### 5. Integração com Chat (RAG Injection)

#### 5.1 Modificar API de Chat
```typescript
// src/app/api/ai/chat/route.ts
// Adicionar imports:
import { searchKnowledgeBase, formatContextFromResults } from '@/lib/knowledge/search';

// No handler POST, antes de chamar streamText:
export async function POST(req: Request) {
  try {
    const userId = await validateUserAuthentication();
    const parsed = BodySchema.safeParse(await req.json());

    if (!parsed.success) {
      return NextResponse.json({ error: 'Corpo da requisição inválido', issues: parsed.error.flatten() }, { status: 400 });
    }

    const { provider, model, messages, temperature = 0.4, attachments } = parsed.data;

    // ... validações existentes ...

    // ========== RAG: Buscar contexto relevante ==========
    let contextMessage: ChatMessage | null = null;

    try {
      // Pegar última mensagem do usuário
      const lastUserMessage = messages.filter(m => m.role === 'user').pop();

      if (lastUserMessage) {
        const dbUser = await getUserFromClerkId(userId);

        // Buscar chunks relevantes (multi-tenant)
        const searchResults = await searchKnowledgeBase(lastUserMessage.content, {
          userId: dbUser.id,
          // workspaceId: obter do contexto se aplicável
          topK: 5,
        });

        // Formatar contexto
        if (searchResults.length > 0) {
          const contextText = formatContextFromResults(searchResults);
          contextMessage = {
            role: 'system',
            content: `Use o seguinte contexto da base de conhecimento SOMENTE se for relevante para responder à pergunta do usuário. Se o contexto não for útil, ignore-o e responda com base no seu conhecimento geral.

<context>
${contextText}
</context>

Instruções:
- Cite as fontes quando usar informações do contexto
- Se o contexto for insuficiente ou irrelevante, não force seu uso
- Mantenha respostas claras e objetivas`,
          };
        }
      }
    } catch (ragError) {
      // Falha no RAG não deve bloquear o chat
      console.error('Erro ao buscar contexto RAG:', ragError);
    }
    // ====================================================

    // Merge de mensagens com contexto e anexos
    let mergedMessages: ChatMessage[] = messages as ChatMessage[];

    // Adicionar contexto RAG no início (se houver)
    if (contextMessage) {
      mergedMessages = [contextMessage, ...mergedMessages];
    }

    // Adicionar anexos no final (código existente)
    if (attachments && attachments.length > 0) {
      const lines = attachments.map(a => `- ${a.name}: ${a.url}`).join('\n');
      const attachNote = `Anexos:\n${lines}`;
      mergedMessages = [...mergedMessages, { role: 'user' as const, content: attachNote }];
    }

    // ... resto do código (créditos, streamText, etc.) ...
  } catch {
    return NextResponse.json({ error: 'Erro interno do servidor' }, { status: 500 });
  }
}
```

---

### 6. Frontend - Admin Panel

#### 6.1 Nova Rota Admin
**Caminho**: `/admin/knowledge`

```typescript
// src/app/admin/knowledge/page.tsx
'use client';

import { useState } from 'react';
import { useKnowledgeEntries } from '@/hooks/admin/use-admin-knowledge';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { KnowledgeEntryForm } from '@/components/admin/knowledge-entry-form';
import { KnowledgeEntryList } from '@/components/admin/knowledge-entry-list';
import { Plus, Search } from 'lucide-react';

export default function KnowledgeBasePage() {
  const [isFormOpen, setIsFormOpen] = useState(false);
  const [selectedEntryId, setSelectedEntryId] = useState<string | null>(null);
  const [searchTerm, setSearchTerm] = useState('');
  const [statusFilter, setStatusFilter] = useState<'ACTIVE' | 'DRAFT' | 'ARCHIVED' | null>(null);

  const { data, isLoading } = useKnowledgeEntries({
    search: searchTerm,
    status: statusFilter,
  });

  return (
    <div className="space-y-6 p-8">
      <div className="flex justify-between items-center">
        <h1 className="text-3xl font-bold">Base de Conhecimento</h1>
        <Button onClick={() => setIsFormOpen(true)}>
          <Plus className="w-4 h-4 mr-2" />
          Nova Entrada
        </Button>
      </div>

      {/* Filtros */}
      <div className="flex gap-4">
        <div className="relative flex-1">
          <Search className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-muted-foreground" />
          <Input
            placeholder="Buscar por título ou tags..."
            value={searchTerm}
            onChange={(e) => setSearchTerm(e.target.value)}
            className="pl-10"
          />
        </div>
        <select
          value={statusFilter || ''}
          onChange={(e) => setStatusFilter(e.target.value as any || null)}
          className="border rounded-md px-4"
        >
          <option value="">Todos os status</option>
          <option value="ACTIVE">Ativo</option>
          <option value="DRAFT">Rascunho</option>
          <option value="ARCHIVED">Arquivado</option>
        </select>
      </div>

      {/* Listagem */}
      <KnowledgeEntryList
        entries={data?.entries || []}
        isLoading={isLoading}
        onEdit={(id) => {
          setSelectedEntryId(id);
          setIsFormOpen(true);
        }}
      />

      {/* Modal de Criação/Edição */}
      {isFormOpen && (
        <KnowledgeEntryForm
          entryId={selectedEntryId}
          onClose={() => {
            setIsFormOpen(false);
            setSelectedEntryId(null);
          }}
        />
      )}
    </div>
  );
}
```

#### 6.2 Componente de Formulário
```typescript
// src/components/admin/knowledge-entry-form.tsx
'use client';

import { useEffect } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import {
  useCreateKnowledgeEntry,
  useUpdateKnowledgeEntry,
  useKnowledgeEntry,
} from '@/hooks/admin/use-admin-knowledge';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Dialog, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog';
import { toast } from 'sonner';

const EntrySchema = z.object({
  title: z.string().min(1, 'Título obrigatório').max(500),
  content: z.string().min(1, 'Conteúdo obrigatório').max(50000),
  tags: z.string(), // Será convertido para array
  status: z.enum(['ACTIVE', 'DRAFT', 'ARCHIVED']),
});

type FormData = z.infer<typeof EntrySchema>;

interface Props {
  entryId?: string | null;
  onClose: () => void;
}

export function KnowledgeEntryForm({ entryId, onClose }: Props) {
  const isEditing = !!entryId;
  const { data: entry } = useKnowledgeEntry(entryId);
  const createMutation = useCreateKnowledgeEntry();
  const updateMutation = useUpdateKnowledgeEntry();

  const form = useForm<FormData>({
    resolver: zodResolver(EntrySchema),
    defaultValues: {
      title: '',
      content: '',
      tags: '',
      status: 'ACTIVE',
    },
  });

  useEffect(() => {
    if (entry) {
      form.reset({
        title: entry.title,
        content: entry.content,
        tags: entry.tags.join(', '),
        status: entry.status,
      });
    }
  }, [entry, form]);

  const onSubmit = async (data: FormData) => {
    const tags = data.tags
      .split(',')
      .map((t) => t.trim())
      .filter(Boolean);

    try {
      if (isEditing && entryId) {
        await updateMutation.mutateAsync({
          id: entryId,
          data: { ...data, tags },
        });
        toast.success('Entrada atualizada e reindexada com sucesso!');
      } else {
        await createMutation.mutateAsync({ ...data, tags });
        toast.success('Entrada criada e indexada com sucesso!');
      }
      onClose();
    } catch (error) {
      toast.error('Erro ao salvar entrada');
    }
  };

  return (
    <Dialog open onOpenChange={onClose}>
      <DialogContent className="max-w-3xl max-h-[90vh] overflow-y-auto">
        <DialogHeader>
          <DialogTitle>
            {isEditing ? 'Editar Entrada' : 'Nova Entrada'}
          </DialogTitle>
        </DialogHeader>

        <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
          <div>
            <label className="block text-sm font-medium mb-2">Título</label>
            <Input {...form.register('title')} placeholder="Título da entrada" />
            {form.formState.errors.title && (
              <p className="text-sm text-red-500 mt-1">
                {form.formState.errors.title.message}
              </p>
            )}
          </div>

          <div>
            <label className="block text-sm font-medium mb-2">
              Conteúdo (Markdown/Texto)
            </label>
            <Textarea
              {...form.register('content')}
              rows={15}
              placeholder="Digite o conteúdo da base de conhecimento..."
              className="font-mono text-sm"
            />
            {form.formState.errors.content && (
              <p className="text-sm text-red-500 mt-1">
                {form.formState.errors.content.message}
              </p>
            )}
          </div>

          <div>
            <label className="block text-sm font-medium mb-2">
              Tags (separadas por vírgula)
            </label>
            <Input
              {...form.register('tags')}
              placeholder="ex: documentação, api, guia"
            />
          </div>

          <div>
            <label className="block text-sm font-medium mb-2">Status</label>
            <select {...form.register('status')} className="w-full border rounded-md px-3 py-2">
              <option value="ACTIVE">Ativo</option>
              <option value="DRAFT">Rascunho</option>
              <option value="ARCHIVED">Arquivado</option>
            </select>
          </div>

          <div className="flex gap-2 justify-end">
            <Button type="button" variant="outline" onClick={onClose}>
              Cancelar
            </Button>
            <Button
              type="submit"
              disabled={createMutation.isPending || updateMutation.isPending}
            >
              {isEditing ? 'Atualizar e Reindexar' : 'Criar e Indexar'}
            </Button>
          </div>
        </form>
      </DialogContent>
    </Dialog>
  );
}
```

#### 6.3 Componente de Listagem
```typescript
// src/components/admin/knowledge-entry-list.tsx
'use client';

import { Button } from '@/components/ui/button';
import { useDeleteKnowledgeEntry, useReindexEntry } from '@/hooks/admin/use-admin-knowledge';
import { Edit, Trash2, RefreshCw } from 'lucide-react';
import { toast } from 'sonner';

interface Entry {
  id: string;
  title: string;
  tags: string[];
  status: string;
  chunks: any[];
  updatedAt: string;
}

interface Props {
  entries: Entry[];
  isLoading: boolean;
  onEdit: (id: string) => void;
}

export function KnowledgeEntryList({ entries, isLoading, onEdit }: Props) {
  const deleteMutation = useDeleteKnowledgeEntry();
  const reindexMutation = useReindexEntry();

  const handleDelete = async (id: string, title: string) => {
    if (!confirm(`Tem certeza que deseja deletar "${title}"?`)) return;

    try {
      await deleteMutation.mutateAsync(id);
      toast.success('Entrada deletada com sucesso!');
    } catch {
      toast.error('Erro ao deletar entrada');
    }
  };

  const handleReindex = async (id: string) => {
    try {
      await reindexMutation.mutateAsync(id);
      toast.success('Reindexação concluída!');
    } catch {
      toast.error('Erro ao reindexar entrada');
    }
  };

  if (isLoading) {
    return <div className="text-center py-8">Carregando...</div>;
  }

  if (entries.length === 0) {
    return (
      <div className="text-center py-12 text-muted-foreground">
        Nenhuma entrada encontrada. Crie a primeira!
      </div>
    );
  }

  return (
    <div className="space-y-4">
      {entries.map((entry) => (
        <div
          key={entry.id}
          className="border rounded-lg p-4 hover:bg-accent/50 transition-colors"
        >
          <div className="flex justify-between items-start">
            <div className="flex-1">
              <h3 className="font-semibold text-lg">{entry.title}</h3>
              <div className="flex gap-2 mt-2">
                {entry.tags.map((tag) => (
                  <span
                    key={tag}
                    className="px-2 py-1 bg-primary/10 text-primary text-xs rounded-full"
                  >
                    {tag}
                  </span>
                ))}
              </div>
              <p className="text-sm text-muted-foreground mt-2">
                {entry.chunks.length} chunks • Status: {entry.status}
              </p>
            </div>

            <div className="flex gap-2">
              <Button
                size="sm"
                variant="ghost"
                onClick={() => handleReindex(entry.id)}
                disabled={reindexMutation.isPending}
              >
                <RefreshCw className="w-4 h-4" />
              </Button>
              <Button size="sm" variant="ghost" onClick={() => onEdit(entry.id)}>
                <Edit className="w-4 h-4" />
              </Button>
              <Button
                size="sm"
                variant="ghost"
                onClick={() => handleDelete(entry.id, entry.title)}
                disabled={deleteMutation.isPending}
              >
                <Trash2 className="w-4 h-4 text-red-500" />
              </Button>
            </div>
          </div>
        </div>
      ))}
    </div>
  );
}
```

#### 6.4 Custom Hooks
```typescript
// src/hooks/admin/use-admin-knowledge.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '@/lib/api-client';

interface KnowledgeEntry {
  id: string;
  title: string;
  content: string;
  tags: string[];
  status: 'ACTIVE' | 'DRAFT' | 'ARCHIVED';
  chunks: any[];
  updatedAt: string;
}

interface CreateEntryData {
  title: string;
  content: string;
  tags: string[];
  status?: 'ACTIVE' | 'DRAFT' | 'ARCHIVED';
}

export function useKnowledgeEntries(params?: { search?: string; status?: string }) {
  const queryString = new URLSearchParams(params as any).toString();
  return useQuery<{ entries: KnowledgeEntry[]; pagination: any }>({
    queryKey: ['admin', 'knowledge', params],
    queryFn: () => api.get(`/api/admin/knowledge?${queryString}`),
    staleTime: 30_000,
  });
}

export function useKnowledgeEntry(id?: string | null) {
  return useQuery<KnowledgeEntry>({
    queryKey: ['admin', 'knowledge', id],
    queryFn: () => api.get(`/api/admin/knowledge/${id}`),
    enabled: !!id,
  });
}

export function useCreateKnowledgeEntry() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: CreateEntryData) => api.post('/api/admin/knowledge', data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['admin', 'knowledge'] });
    },
  });
}

export function useUpdateKnowledgeEntry() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<CreateEntryData> }) =>
      api.put(`/api/admin/knowledge/${id}`, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['admin', 'knowledge'] });
    },
  });
}

export function useDeleteKnowledgeEntry() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: string) => api.delete(`/api/admin/knowledge/${id}`),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['admin', 'knowledge'] });
    },
  });
}

export function useReindexEntry() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: string) => api.post(`/api/admin/knowledge/${id}/reindex`, {}),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['admin', 'knowledge'] });
    },
  });
}
```

---

## 7. Configuração e Deploy

### 7.1 Criar Index no Upstash
1. Acessar dashboard Upstash: https://console.upstash.com/
2. Criar novo Vector Database
3. Configurar:
   - **Name**: knowledge-base-prod
   - **Dimensions**: 1536 (OpenAI text-embedding-3-small)
   - **Similarity**: Cosine
4. Copiar `UPSTASH_VECTOR_REST_URL` e `UPSTASH_VECTOR_REST_TOKEN`
5. Adicionar ao `.env.local`

### 7.2 Atualizar .env.example
```env
# Upstash Vector (RAG Knowledge Base)
UPSTASH_VECTOR_REST_URL=https://your-db.upstash.io
UPSTASH_VECTOR_REST_TOKEN=your_token_here

# RAG Configuration
RAG_TOP_K=5
RAG_CHUNK_SIZE=800
RAG_CHUNK_OVERLAP=100
RAG_MAX_CONTEXT_TOKENS=2000
```

### 7.3 Migrations
```bash
npx prisma db push  # Aplicar schema
npx prisma generate # Gerar client
```

### 7.4 Adicionar ao Sidebar Admin
```typescript
// src/components/app/sidebar.tsx
// Adicionar item:
{
  label: 'Base de Conhecimento',
  href: '/admin/knowledge',
  icon: BookOpen,
}
```

---

## 8. Segurança & Multi-tenancy

### 8.1 Isolamento de Dados
- **Filtros no Upstash**: Sempre incluir `userId` ou `workspaceId` nos filtros de busca
- **Validação Server-side**: Verificar ownership antes de updates/deletes
- **Metadata**: Armazenar tenant info nos vetores para filtragem

### 8.2 Sanitização
- **HTML/XSS**: Sanitizar conteúdo antes de exibir (usar DOMPurify se necessário)
- **Logs**: Não logar conteúdo sensível, apenas IDs e métricas

### 8.3 Rate Limiting
- Considerar rate limit em endpoints de indexação (pesado)
- Admin-only: Apenas admins podem criar/editar entradas

---

## 9. Melhorias Futuras

### 9.1 Funcionalidades Avançadas
- **Upload de arquivos**: Permitir upload de .txt/.md e extrair conteúdo
- **Preview**: Visualizar markdown renderizado
- **Versionamento**: Histórico de alterações em entradas
- **Analytics**: Métricas de uso (queries mais comuns, chunks mais recuperados)

### 9.2 Performance
- **Batch Processing**: Processar múltiplos documentos em background (BullMQ)
- **Cache**: Cache de embeddings para queries frequentes
- **Reranking**: Adicionar modelo de reranking para melhor relevância

### 9.3 UX
- **Highlight**: Destacar trechos relevantes nas respostas do chat
- **Feedback**: Permitir thumbs up/down em respostas para melhorar contexto
- **Fontes**: Exibir fontes citadas na interface do chat

---

## 10. Checklist de Implementação

### Backend
- [ ] Instalar `@upstash/vector`
- [ ] Criar modelos Prisma (`KnowledgeBaseEntry`, `KnowledgeChunk`)
- [ ] Configurar Upstash Vector client (`vector-client.ts`)
- [ ] Implementar embeddings com OpenRouter (`embeddings.ts`)
- [ ] Implementar chunking (`chunking.ts`)
- [ ] Implementar indexer (`indexer.ts`)
- [ ] Implementar search (`search.ts`)
- [ ] Criar API routes CRUD (`/api/admin/knowledge/...`)
- [ ] Integrar RAG no chat route (`/api/ai/chat/route.ts`)

### Frontend
- [ ] Criar página admin (`/admin/knowledge/page.tsx`)
- [ ] Criar formulário de entrada (`knowledge-entry-form.tsx`)
- [ ] Criar listagem (`knowledge-entry-list.tsx`)
- [ ] Criar custom hooks TanStack Query (`use-admin-knowledge.ts`)
- [ ] Adicionar item no sidebar admin

### Configuração
- [ ] Criar index no Upstash console
- [ ] Adicionar variáveis de ambiente (`.env.local` e `.env.example`)
- [ ] Rodar migrations (`npx prisma db push`)
- [ ] Testar criação/edição/deleção de entradas
- [ ] Testar busca RAG no chat

### Testes
- [ ] Criar 2-3 entradas de teste com conteúdos distintos
- [ ] Testar busca RAG com queries relacionadas
- [ ] Validar isolamento multi-tenant
- [ ] Testar reindexação manual
- [ ] Simular falha do Upstash (verificar fallback)

### Documentação
- [ ] Atualizar README com instruções RAG
- [ ] Documentar fluxo de indexação
- [ ] Documentar configuração Upstash

---

## 11. Custos & Estimativas

### Upstash Vector
- **Free Tier**: 10k vetores, 10k queries/dia
- **Pricing**: $0.40/100k vectors/mês após free tier
- **Estimativa**: 100 documentos ≈ 500-1000 chunks ≈ bem dentro do free tier

### OpenRouter Embeddings
- **text-embedding-3-small**: $0.02 / 1M tokens
- **Estimativa**: 100 documentos (~500 chunks de 800 chars) ≈ 100k tokens ≈ $0.002

### Viabilidade
- MVP: Totalmente gratuito no free tier
- Produção: Custo muito baixo (<$5/mês para 1000+ documentos)

---

## Referências

- **Upstash Vector Docs**: https://upstash.com/docs/vector/overall/getstarted
- **Upstash Vector TS SDK**: https://upstash.com/docs/vector/sdks/ts/getting-started
- **Vercel AI SDK Embeddings**: https://sdk.vercel.ai/docs/ai-sdk-core/embeddings
- **OpenRouter API**: https://openrouter.ai/docs
- **Prompt de Referência**: `/prompts/upstash-knowledge-base.md`

---

## Notas Importantes

1. **Somente TXT e MD**: Removido parsing de PDF/DOCX para simplificar MVP
2. **OpenRouter**: Usar OpenRouter para embeddings (já configurado no projeto)
3. **Multi-tenant**: Implementar desde o início com userId/workspaceId
4. **Fallback**: Chat deve continuar funcionando se Upstash falhar
5. **Admin-only**: Apenas admins criam/editam base de conhecimento
6. **Observabilidade**: Logs sem conteúdo sensível (apenas IDs, tempos, contagens)
