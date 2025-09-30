# Prompt: Sistema de Templates e Criativos Unificado - Studio Lagosta V2

## Start Here
- Leia CLAUDE.md para padrões do repositório e arquitetura do Studio Lagosta V2.
- Consulte o mapeamento detalhado de funcionalidades fornecido pelo usuário.
- Este prompt implementa um sistema de criação de templates e geração de criativos com **renderização unificada** usando HTML5 Canvas tanto no frontend quanto no backend, eliminando inconsistências entre preview e geração final.

## Contexto e Problema

### Sistema Original (Studio Lagosta V1)
O Studio Lagosta original possui:
- ✅ Sistema robusto de templates com layers (texto, imagem, gradiente, logo, elemento)
- ✅ Editor visual drag-and-drop com 2342 linhas de código
- ✅ RenderEngine unificado (`shared/renderEngine.ts` - 2217 linhas)
- ❌ **INCONSISTÊNCIA**: Editor usa DOM/CSS, geração usa Canvas (@napi-rs/canvas)
- ❌ **RESULTADO**: O que o usuário vê no editor ≠ imagem final gerada

### Causas da Inconsistência
1. **Texto**: CSS word-wrap ≠ Canvas measureText() + algoritmo manual
2. **Fontes**: Web fonts (CSS) ≠ Fontes registradas (GlobalFonts)
3. **Gradientes**: CSS linear-gradient() ≠ Canvas createLinearGradient()
4. **Line height**: CSS line-height ≠ Cálculo manual (fontSize * lineHeight)

### Solução Proposta
**Sistema Híbrido Unificado:**
- **Edição**: DOM com drag-and-drop para UX (interatividade)
- **Preview**: HTML5 Canvas usando RenderEngine (preview real-time)
- **Geração Backend**: Node.js Canvas usando o MESMO RenderEngine
- **Resultado**: Preview = Geração Final (100% consistência)

## Objetivo
Implementar no **Studio Lagosta V2** (Next.js 15 + Prisma + Clerk) um sistema completo de criação e gestão de templates/criativos com renderização unificada, baseado na arquitetura do V1 mas modernizado e sem inconsistências.

## Escopo

### Funcionalidades Principais
1. **Projetos**: CRUD completo de projetos (já implementado parcialmente no V2)
2. **Templates**: Editor visual com sistema de camadas (layers)
3. **Geração de Criativos**: Single e Carousel com preview em tempo real
4. **Renderização Unificada**: Canvas API em frontend e backend
5. **Gestão de Assets**: Logos, elementos, fontes customizadas
6. **Integrações**: Google Drive (fotos), Vercel Blob (storage)

### Fora do Escopo (Fase 2)
- Publicação no Instagram/Facebook
- Sistema de cupons/descontos
- Analytics avançados
- Templates marketplace
- IA para geração de textos

## Requisitos de Configuração

### Variáveis de Ambiente
Adicione em `.env.local`:
```env
# Vercel Blob (Storage de imagens)
BLOB_READ_WRITE_TOKEN=seu_token_blob
BLOB_MAX_SIZE_MB=25

# Google Drive API (Opcional - importação de fotos)
GOOGLE_DRIVE_CLIENT_ID=seu_client_id
GOOGLE_DRIVE_CLIENT_SECRET=seu_client_secret
GOOGLE_DRIVE_REFRESH_TOKEN=seu_refresh_token

# Fontes (Sistema de fontes customizadas)
FONTS_DIR=/tmp/fonts # Diretório temporário para fontes registradas
```

### Dependências
```bash
npm install @napi-rs/canvas
npm install sharp # Processamento de imagens
npm install fitty # Auto-resize de texto (opcional frontend)
```

### Documentação de Referência
- **Node.js Canvas**: https://github.com/Automattic/node-canvas
- **@napi-rs/canvas**: https://github.com/Brooooooklyn/canvas (mais rápido, usado no V1)
- **Vercel Blob**: https://vercel.com/docs/storage/vercel-blob
- **HTML5 Canvas API**: https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API

## Arquitetura de Alto Nível

### Stack Tecnológica
```
Frontend:
  - Next.js 15 (App Router)
  - React 18 + TypeScript
  - Tailwind CSS v4 + Radix UI
  - TanStack Query (data fetching)
  - React Hook Form + Zod (validação)
  - HTML5 Canvas API (preview unificado)

Backend:
  - Next.js API Routes (server-side)
  - @napi-rs/canvas (renderização)
  - Prisma ORM (PostgreSQL)
  - Clerk Auth (autenticação)
  - Vercel Blob SDK (storage)

Compartilhado:
  - RenderEngine unificado (lib/render-engine.ts)
  - Font Config (lib/font-config.ts)
  - Type definitions (types/template.ts)
```

### Hierarquia de Dados
```
User (Clerk)
  └── Projects
       ├── Templates
       │    ├── Design Data (JSON)
       │    │    ├── Canvas config (width, height, background)
       │    │    └── Layers array
       │    │         ├── Text layers (content, style, textboxConfig)
       │    │         ├── Image layers (fileUrl, crop, filters)
       │    │         ├── Gradient layers (type, stops, angle)
       │    │         ├── Logo layers (logoId, fileUrl)
       │    │         └── Element layers (elementId, fileUrl)
       │    ├── Dynamic Fields (campos editáveis)
       │    └── Thumbnail URL
       └── Generations (Criativos)
            ├── Template usado
            ├── Field Values (dados preenchidos)
            ├── Result URL (imagem final)
            └── Metadata (autor, data, projeto)
```

## Modelagem de Dados (Prisma Schema)

### Schema Proposto

```prisma
// prisma/schema.prisma

// Projeto: container de templates e criativos
model Project {
  id          Int       @id @default(autoincrement())
  name        String
  description String?   @db.Text
  status      ProjectStatus @default(ACTIVE)
  logoUrl     String?

  // Integrações (opcional)
  googleDriveFolderId String?
  makeWebhookAnalyzeUrl String?
  makeWebhookCreativeUrl String?

  // Relações
  userId      String    // Clerk user ID
  workspaceId Int?      // Multi-tenant (futuro)

  templates   Template[]
  generations Generation[]
  logos       Logo[]
  elements    Element[]
  fonts       CustomFont[]

  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@index([userId])
  @@index([workspaceId])
}

enum ProjectStatus {
  ACTIVE
  INACTIVE
  ARCHIVED
}

// Template: modelo reutilizável
model Template {
  id          Int       @id @default(autoincrement())
  name        String
  type        TemplateType
  dimensions  String    // "1080x1920", "1080x1350", "1080x1080"

  // Design data (JSON com layers)
  designData  Json      // { canvas: {...}, layers: [...] }

  // Campos dinâmicos (definição)
  dynamicFields Json    @default("[]") // [{ layerId, fieldType, label }]

  // Thumbnail
  thumbnailUrl String?

  // Relações
  projectId   Int
  project     Project   @relation(fields: [projectId], references: [id], onDelete: Cascade)

  generations Generation[]

  createdBy   String    // Clerk user ID
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@index([projectId])
  @@index([type])
  @@index([createdBy])
}

enum TemplateType {
  STORY      // 1080x1920 (9:16)
  FEED       // 1080x1350 (4:5)
  SQUARE     // 1080x1080 (1:1)
}

// Generation: criativo gerado
model Generation {
  id          String    @id @default(cuid())
  status      GenerationStatus @default(PROCESSING)

  // Template usado
  templateId  Int
  template    Template  @relation(fields: [templateId], references: [id], onDelete: Restrict)

  // Dados preenchidos
  fieldValues Json      // { "layer-id": "value", ... }

  // Resultado
  resultUrl   String?

  // Metadata
  projectId   Int
  project     Project   @relation(fields: [projectId], references: [id], onDelete: Cascade)

  authorName  String?
  templateName String?
  projectName String?

  createdBy   String    // Clerk user ID
  createdAt   DateTime  @default(now())
  completedAt DateTime?

  @@index([projectId])
  @@index([templateId])
  @@index([status])
  @@index([createdBy])
  @@index([createdAt])
}

enum GenerationStatus {
  PROCESSING
  COMPLETED
  FAILED
}

// Logo: logos do projeto
model Logo {
  id          Int       @id @default(autoincrement())
  name        String
  fileUrl     String

  projectId   Int
  project     Project   @relation(fields: [projectId], references: [id], onDelete: Cascade)

  uploadedBy  String    // Clerk user ID
  createdAt   DateTime  @default(now())

  @@index([projectId])
}

// Element: elementos gráficos reutilizáveis
model Element {
  id          Int       @id @default(autoincrement())
  name        String
  fileUrl     String
  category    String?   // "icons", "illustrations", "shapes"

  projectId   Int
  project     Project   @relation(fields: [projectId], references: [id], onDelete: Cascade)

  uploadedBy  String    // Clerk user ID
  createdAt   DateTime  @default(now())

  @@index([projectId])
  @@index([category])
}

// CustomFont: fontes customizadas
model CustomFont {
  id          Int       @id @default(autoincrement())
  name        String
  fontFamily  String
  fileUrl     String    // URL do arquivo .ttf/.otf

  projectId   Int
  project     Project   @relation(fields: [projectId], references: [id], onDelete: Cascade)

  uploadedBy  String    // Clerk user ID
  createdAt   DateTime  @default(now())

  @@index([projectId])
}
```

### Estrutura do designData (JSON)

```typescript
// types/template.ts

interface DesignData {
  canvas: {
    width: number;       // 1080
    height: number;      // 1920, 1350, ou 1080
    backgroundColor: string; // "#FFFFFF" ou "transparent"
  };

  layers: Layer[];
}

interface Layer {
  id: string;          // UUID único
  type: LayerType;
  name: string;        // Nome exibido
  visible: boolean;    // Visibilidade
  locked: boolean;     // Bloqueio de edição
  order: number;       // Z-index (0 = fundo, 999 = topo)

  // Posição e tamanho (sempre em coordenadas absolutas 1:1)
  position: { x: number; y: number };
  size: { width: number; height: number };
  rotation?: number;   // Em graus (0-360)

  // Conteúdo (depende do tipo)
  content?: string;

  // Estilo (depende do tipo)
  style?: LayerStyle;

  // Campo dinâmico?
  isDynamic?: boolean;

  // Configurações específicas de texto
  textboxConfig?: TextboxConfig;

  // Referências para assets
  logoId?: number;
  elementId?: number;
  fileUrl?: string;

  // Parent layer (para agrupamento futuro)
  parentId?: string | null;
}

enum LayerType {
  TEXT = 'text',
  IMAGE = 'image',
  GRADIENT = 'gradient',
  GRADIENT2 = 'gradient2',
  LOGO = 'logo',
  ELEMENT = 'element',
}

interface LayerStyle {
  // Texto
  fontSize?: number;
  fontFamily?: string;
  fontWeight?: string | number;
  fontStyle?: 'normal' | 'italic';
  color?: string;
  textAlign?: 'left' | 'center' | 'right';
  textTransform?: 'none' | 'uppercase' | 'lowercase' | 'capitalize';
  letterSpacing?: number;
  lineHeight?: number;

  // Gradiente
  gradientType?: 'linear' | 'radial';
  gradientAngle?: number;
  gradientStops?: Array<{ color: string; position: number }>;

  // Imagem
  objectFit?: 'contain' | 'cover' | 'fill';
  opacity?: number;
  filter?: string; // CSS filter string

  // Efeitos
  shadow?: {
    offsetX: number;
    offsetY: number;
    blur: number;
    color: string;
  };
  border?: {
    width: number;
    color: string;
    radius: number;
  };
}

interface TextboxConfig {
  spacing?: number;        // Espaçamento entre linhas (em pixels)
  anchor?: 'top' | 'middle' | 'bottom'; // Ancoragem do texto na caixa
  textMode?: TextMode;

  // Auto-resize (ajusta fonte automaticamente)
  autoResize?: {
    minFontSize: number;
    maxFontSize: number;
  };

  // Auto-wrap (quebra de linha automática)
  autoWrap?: {
    lineHeight: number;    // Multiplicador (1.2 = 120%)
    breakMode: 'word' | 'char' | 'hybrid';
    autoExpand: boolean;   // Expande caixa se necessário
  };

  wordBreak?: boolean;     // Permite quebra de palavra
}

enum TextMode {
  AUTO_RESIZE_SINGLE = 'auto-resize-single',   // Uma linha, ajusta fonte
  AUTO_RESIZE_MULTI = 'auto-resize-multi',     // Múltiplas linhas, ajusta fonte
  AUTO_WRAP_FIXED = 'auto-wrap-fixed',         // Fonte fixa, quebra linhas
  FITTY = 'fitty',                             // Usa biblioteca fitty.js
}

interface DynamicField {
  layerId: string;       // ID da layer
  fieldType: 'text' | 'image' | 'color' | 'fontSize';
  label: string;         // Label no formulário
  placeholder?: string;
  defaultValue?: any;
  required?: boolean;
  validation?: {
    min?: number;
    max?: number;
    pattern?: string;
  };
}
```

## APIs (Rotas Propostas)

### Projetos

```typescript
// src/app/api/projects/route.ts

// GET /api/projects - Listar projetos do usuário
export async function GET(request: Request) {
  const { userId } = await auth();
  if (!userId) return unauthorized();

  const projects = await db.project.findMany({
    where: { userId },
    include: {
      _count: {
        select: { templates: true, generations: true }
      }
    },
    orderBy: { updatedAt: 'desc' }
  });

  return json(projects);
}

// POST /api/projects - Criar projeto
export async function POST(request: Request) {
  const { userId } = await auth();
  if (!userId) return unauthorized();

  const body = await request.json();
  const validated = createProjectSchema.parse(body);

  const project = await db.project.create({
    data: {
      ...validated,
      userId,
      status: 'ACTIVE',
    }
  });

  return json(project, { status: 201 });
}
```

```typescript
// src/app/api/projects/[id]/route.ts

// GET /api/projects/:id - Detalhes do projeto
// PUT /api/projects/:id - Atualizar projeto
// DELETE /api/projects/:id - Deletar projeto
```

### Templates

```typescript
// src/app/api/projects/[projectId]/templates/route.ts

// GET /api/projects/:projectId/templates - Listar templates
export async function GET(
  request: Request,
  { params }: { params: { projectId: string } }
) {
  const { userId } = await auth();
  if (!userId) return unauthorized();

  const projectId = parseInt(params.projectId);

  // Verificar ownership
  const project = await db.project.findFirst({
    where: { id: projectId, userId }
  });

  if (!project) return notFound();

  const templates = await db.template.findMany({
    where: { projectId },
    orderBy: { updatedAt: 'desc' }
  });

  return json(templates);
}

// POST /api/projects/:projectId/templates - Criar template
export async function POST(
  request: Request,
  { params }: { params: { projectId: string } }
) {
  const { userId } = await auth();
  if (!userId) return unauthorized();

  const body = await request.json();
  const validated = createTemplateSchema.parse(body);

  // Criar template com layers padrão
  const defaultLayers = getDefaultLayersForType(validated.type);

  const template = await db.template.create({
    data: {
      ...validated,
      projectId: parseInt(params.projectId),
      createdBy: userId,
      designData: {
        canvas: getDefaultCanvas(validated.type),
        layers: defaultLayers,
      },
      dynamicFields: [],
    }
  });

  return json(template, { status: 201 });
}
```

```typescript
// src/app/api/templates/[id]/route.ts

// GET /api/templates/:id - Detalhes do template
// PUT /api/templates/:id - Atualizar template
export async function PUT(
  request: Request,
  { params }: { params: { id: string } }
) {
  const { userId } = await auth();
  if (!userId) return unauthorized();

  const templateId = parseInt(params.id);
  const body = await request.json();
  const validated = updateTemplateSchema.parse(body);

  // Verificar ownership
  const template = await db.template.findFirst({
    where: { id: templateId },
    include: { project: true }
  });

  if (!template || template.project.userId !== userId) {
    return notFound();
  }

  const updated = await db.template.update({
    where: { id: templateId },
    data: validated,
  });

  return json(updated);
}

// DELETE /api/templates/:id - Deletar template
```

```typescript
// src/app/api/templates/[id]/thumbnail/route.ts

// POST /api/templates/:id/thumbnail - Gerar thumbnail
export async function POST(
  request: Request,
  { params }: { params: { id: string } }
) {
  const { userId } = await auth();
  if (!userId) return unauthorized();

  const templateId = parseInt(params.id);

  // Carregar template
  const template = await db.template.findFirst({
    where: { id: templateId },
    include: { project: true }
  });

  if (!template || template.project.userId !== userId) {
    return notFound();
  }

  // Gerar thumbnail (800x1000 para 4:5 ratio)
  const thumbnailUrl = await generateThumbnail(template);

  // Atualizar template
  await db.template.update({
    where: { id: templateId },
    data: { thumbnailUrl }
  });

  return json({ thumbnailUrl });
}
```

### Gerações (Criativos)

```typescript
// src/app/api/projects/[projectId]/generations/route.ts

// GET /api/projects/:projectId/generations - Listar criativos
export async function GET(
  request: Request,
  { params }: { params: { projectId: string } }
) {
  const { userId } = await auth();
  if (!userId) return unauthorized();

  const { searchParams } = new URL(request.url);
  const page = parseInt(searchParams.get('page') || '1');
  const pageSize = parseInt(searchParams.get('pageSize') || '18');

  const projectId = parseInt(params.projectId);

  // Verificar ownership
  const project = await db.project.findFirst({
    where: { id: projectId, userId }
  });

  if (!project) return notFound();

  const [generations, total] = await Promise.all([
    db.generation.findMany({
      where: { projectId },
      include: { template: true },
      orderBy: { createdAt: 'desc' },
      skip: (page - 1) * pageSize,
      take: pageSize,
    }),
    db.generation.count({ where: { projectId } })
  ]);

  return json({
    generations,
    pagination: {
      page,
      pageSize,
      total,
      totalPages: Math.ceil(total / pageSize)
    }
  });
}

// POST /api/projects/:projectId/generations - Gerar criativo (single)
export async function POST(
  request: Request,
  { params }: { params: { projectId: string } }
) {
  const { userId } = await auth();
  if (!userId) return unauthorized();

  const body = await request.json();
  const validated = createGenerationSchema.parse(body);

  const projectId = parseInt(params.projectId);

  // Verificar ownership
  const project = await db.project.findFirst({
    where: { id: projectId, userId }
  });

  if (!project) return notFound();

  // Criar generation job
  const generation = await db.generation.create({
    data: {
      projectId,
      templateId: validated.templateId,
      fieldValues: validated.fieldValues,
      status: 'PROCESSING',
      createdBy: userId,
      authorName: await getUserName(userId),
      projectName: project.name,
    },
    include: { template: true }
  });

  // Processar geração (pode ser assíncrono)
  const resultUrl = await renderGeneration(generation);

  // Atualizar com resultado
  const completed = await db.generation.update({
    where: { id: generation.id },
    data: {
      status: 'COMPLETED',
      resultUrl,
      completedAt: new Date(),
    }
  });

  return json(completed, { status: 201 });
}
```

```typescript
// src/app/api/projects/[projectId]/generations/carousel/route.ts

// POST /api/projects/:projectId/generations/carousel - Gerar carrossel
export async function POST(
  request: Request,
  { params }: { params: { projectId: string } }
) {
  const { userId } = await auth();
  if (!userId) return unauthorized();

  const body = await request.json();
  const validated = createCarouselSchema.parse(body);

  // validated.slides: Array<{ fieldValues: {...} }>

  // Criar múltiplas generations
  const generations = await Promise.all(
    validated.slides.map((slide, index) =>
      db.generation.create({
        data: {
          projectId: parseInt(params.projectId),
          templateId: validated.templateId,
          fieldValues: slide.fieldValues,
          status: 'PROCESSING',
          createdBy: userId,
          // Metadata para identificar carrossel
          projectName: `Carousel ${index + 1}/${validated.slides.length}`,
        }
      })
    )
  );

  // Processar cada slide
  const results = await Promise.all(
    generations.map(async (gen) => {
      const resultUrl = await renderGeneration(gen);
      return db.generation.update({
        where: { id: gen.id },
        data: {
          status: 'COMPLETED',
          resultUrl,
          completedAt: new Date(),
        }
      });
    })
  );

  return json({ generations: results }, { status: 201 });
}
```

```typescript
// src/app/api/generations/[id]/route.ts

// GET /api/generations/:id - Detalhes da geração
// DELETE /api/generations/:id - Deletar criativo
```

```typescript
// src/app/api/generations/[id]/download/route.ts

// GET /api/generations/:id/download - Download do criativo
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const { userId } = await auth();
  if (!userId) return unauthorized();

  const generation = await db.generation.findFirst({
    where: { id: params.id },
    include: { project: true }
  });

  if (!generation || generation.project.userId !== userId) {
    return notFound();
  }

  if (!generation.resultUrl) {
    return json({ error: 'Generation not completed' }, { status: 400 });
  }

  // Redirecionar para URL da imagem ou servir arquivo
  return Response.redirect(generation.resultUrl);
}
```

### Assets (Logos, Elementos, Fontes)

```typescript
// src/app/api/projects/[projectId]/logos/route.ts
// GET, POST - Listar e fazer upload de logos

// src/app/api/projects/[projectId]/elements/route.ts
// GET, POST - Listar e fazer upload de elementos

// src/app/api/projects/[projectId]/fonts/route.ts
// GET, POST - Listar e fazer upload de fontes customizadas

// src/app/api/upload/route.ts
// POST - Upload genérico para Vercel Blob
export async function POST(request: Request) {
  const { userId } = await auth()
  if (!userId) return json({ error: 'Não autorizado' }, { status: 401 })

  const form = await request.formData()
  const file = form.get('file') as File | null
  if (!file) return json({ error: 'Nenhum arquivo' }, { status: 400 })

  const maxMb = Number(process.env.BLOB_MAX_SIZE_MB || '25')
  const maxBytes = Math.max(1, maxMb) * 1024 * 1024
  if (file.size > maxBytes) {
    return json({ error: `Arquivo muito grande (máx ${maxMb}MB)` }, { status: 413 })
  }

  const ext = file.name?.split('.').pop()?.toLowerCase() || 'bin'
  const safeName = file.name?.replace(/[^a-z0-9._-]/gi, '_') || `upload.${ext}`
  const key = `uploads/${userId}/${Date.now()}-${safeName}`

  const token = process.env.BLOB_READ_WRITE_TOKEN
  const uploaded = await put(key, file, { access: 'public', token })

  return json({
    url: uploaded.url,
    pathname: uploaded.pathname,
    contentType: file.type,
    size: file.size,
    name: file.name,
  })
}
```

## Biblioteca de Integração: Render Engine Unificado

### Arquitetura do RenderEngine

O RenderEngine é o **coração do sistema** - responsável por renderizar layers em Canvas de forma idêntica no frontend e backend.

**Arquivo:** `src/lib/render-engine.ts` (baseado no V1)

```typescript
// src/lib/render-engine.ts

import type { Layer, LayerStyle, TextboxConfig } from '@/types/template';

type CanvasContext = CanvasRenderingContext2D; // Browser ou @napi-rs/canvas

export interface RenderOptions {
  scaleFactor?: number;        // Sempre 1 (sistema 1:1)
  imageLoader?: ImageLoader;   // Loader específico do ambiente
  imageCache?: Map<string, any>;
  fontChecker?: FontChecker;
}

export type ImageLoader = (url: string) => Promise<any>;
export type FontChecker = (fontName: string) => Promise<FontValidationResult>;

export interface FontValidationResult {
  isValid: boolean;
  fallbackUsed: boolean;
  fallbackFont?: string;
  confidence: number;
}

export class RenderEngine {
  /**
   * Renderiza uma layer no canvas context
   * Funciona tanto no browser quanto no Node.js
   */
  static async renderLayer(
    ctx: CanvasContext,
    layer: Layer,
    fieldValues: Record<string, any>,
    options: RenderOptions = {}
  ): Promise<void> {
    const {
      scaleFactor = 1,
      imageLoader,
      imageCache,
      fontChecker,
    } = options;

    // Aplicar fieldValues se layer é dinâmica
    const finalLayer = this.applyFieldValues(layer, fieldValues);

    // Pular layers invisíveis
    if (!finalLayer.visible) return;

    // Salvar contexto
    ctx.save();

    // Aplicar transformações (posição, rotação)
    this.applyTransforms(ctx, finalLayer, scaleFactor);

    // Renderizar baseado no tipo
    switch (finalLayer.type) {
      case 'text':
        await this.renderText(ctx, finalLayer, options);
        break;
      case 'image':
        await this.renderImage(ctx, finalLayer, options);
        break;
      case 'gradient':
      case 'gradient2':
        await this.renderGradient(ctx, finalLayer, options);
        break;
      case 'logo':
        await this.renderLogo(ctx, finalLayer, options);
        break;
      case 'element':
        await this.renderElement(ctx, finalLayer, options);
        break;
    }

    // Restaurar contexto
    ctx.restore();
  }

  /**
   * Aplica valores dinâmicos na layer
   */
  private static applyFieldValues(
    layer: Layer,
    fieldValues: Record<string, any>
  ): Layer {
    if (!layer.isDynamic || !fieldValues[layer.id]) {
      return layer;
    }

    const value = fieldValues[layer.id];

    return {
      ...layer,
      content: typeof value === 'string' ? value : layer.content,
      fileUrl: typeof value === 'string' && value.startsWith('http') ? value : layer.fileUrl,
      style: {
        ...layer.style,
        // Permitir override de propriedades de estilo
        ...Object.keys(fieldValues)
          .filter(key => key.startsWith(`${layer.id}_`))
          .reduce((acc, key) => {
            const prop = key.replace(`${layer.id}_`, '');
            acc[prop] = fieldValues[key];
            return acc;
          }, {} as any)
      }
    };
  }

  /**
   * Aplica transformações (posição, rotação)
   */
  private static applyTransforms(
    ctx: CanvasContext,
    layer: Layer,
    scaleFactor: number
  ): void {
    const x = layer.position.x * scaleFactor;
    const y = layer.position.y * scaleFactor;
    const width = layer.size.width * scaleFactor;
    const height = layer.size.height * scaleFactor;

    // Rotação (se aplicável)
    if (layer.rotation && layer.rotation !== 0) {
      const centerX = x + width / 2;
      const centerY = y + height / 2;
      ctx.translate(centerX, centerY);
      ctx.rotate((layer.rotation * Math.PI) / 180);
      ctx.translate(-centerX, -centerY);
    }
  }

  /**
   * Renderiza texto
   */
  private static async renderText(
    ctx: CanvasContext,
    layer: Layer,
    options: RenderOptions
  ): Promise<void> {
    const { fontChecker } = options;
    const style = layer.style || {};
    const content = layer.content || '';

    // Validar fonte
    let fontFamily = style.fontFamily || 'sans-serif';
    if (fontChecker) {
      const validation = await fontChecker(fontFamily);
      if (!validation.isValid && validation.fallbackFont) {
        fontFamily = validation.fallbackFont;
      }
    }

    // Aplicar estilo de fonte
    const fontSize = style.fontSize || 16;
    const fontWeight = style.fontWeight || 'normal';
    const fontStyle = style.fontStyle || 'normal';
    ctx.font = `${fontStyle} ${fontWeight} ${fontSize}px ${fontFamily}`;
    ctx.fillStyle = style.color || '#000000';
    ctx.textAlign = (style.textAlign || 'left') as CanvasTextAlign;

    // Processar texto baseado no textMode
    const textboxConfig = layer.textboxConfig;
    if (textboxConfig) {
      await this.renderTextWithConfig(ctx, layer, content, textboxConfig, options);
    } else {
      // Texto simples (single line)
      const x = layer.position.x;
      const y = layer.position.y + fontSize;
      ctx.fillText(content, x, y);
    }
  }

  /**
   * Renderiza texto com configuração avançada
   */
  private static async renderTextWithConfig(
    ctx: CanvasContext,
    layer: Layer,
    content: string,
    config: TextboxConfig,
    options: RenderOptions
  ): Promise<void> {
    const style = layer.style || {};
    const fontSize = style.fontSize || 16;
    const lineHeight = config.autoWrap?.lineHeight || 1.2;
    const scaledLineHeight = fontSize * lineHeight;

    switch (config.textMode) {
      case 'auto-resize-single':
        // Uma linha, ajusta fonte automaticamente
        await this.renderAutoResizeSingle(ctx, layer, content, config, options);
        break;

      case 'auto-resize-multi':
        // Múltiplas linhas, ajusta fonte automaticamente
        await this.renderAutoResizeMulti(ctx, layer, content, config, options);
        break;

      case 'auto-wrap-fixed':
        // Fonte fixa, quebra linhas automaticamente
        await this.renderAutoWrapFixed(ctx, layer, content, config, options);
        break;

      default:
        // Fallback: renderizar simples
        const x = layer.position.x;
        const y = layer.position.y + fontSize;
        ctx.fillText(content, x, y);
    }
  }

  /**
   * Auto-resize single line
   */
  private static async renderAutoResizeSingle(
    ctx: CanvasContext,
    layer: Layer,
    content: string,
    config: TextboxConfig,
    options: RenderOptions
  ): Promise<void> {
    const style = layer.style || {};
    const maxWidth = layer.size.width;

    const minFontSize = config.autoResize?.minFontSize || 12;
    const maxFontSize = config.autoResize?.maxFontSize || style.fontSize || 16;

    // Binary search para encontrar maior fonte que cabe
    let low = minFontSize;
    let high = maxFontSize;
    let bestFontSize = minFontSize;

    while (low <= high) {
      const mid = Math.floor((low + high) / 2);
      ctx.font = this.buildFontString(mid, style);
      const metrics = ctx.measureText(content);

      if (metrics.width <= maxWidth) {
        bestFontSize = mid;
        low = mid + 1;
      } else {
        high = mid - 1;
      }
    }

    // Renderizar com melhor fonte
    ctx.font = this.buildFontString(bestFontSize, style);
    const x = this.getTextX(layer, ctx.textAlign as string);
    const y = layer.position.y + bestFontSize;
    ctx.fillText(content, x, y);
  }

  /**
   * Auto-resize multi-line
   */
  private static async renderAutoResizeMulti(
    ctx: CanvasContext,
    layer: Layer,
    content: string,
    config: TextboxConfig,
    options: RenderOptions
  ): Promise<void> {
    const style = layer.style || {};
    const maxWidth = layer.size.width;
    const maxHeight = layer.size.height;

    const minFontSize = config.autoResize?.minFontSize || 12;
    const maxFontSize = config.autoResize?.maxFontSize || style.fontSize || 16;
    const lineHeight = config.autoWrap?.lineHeight || 1.2;
    const breakMode = config.autoWrap?.breakMode || 'word';

    // Binary search para encontrar maior fonte que cabe
    let low = minFontSize;
    let high = maxFontSize;
    let bestFontSize = minFontSize;
    let bestLines: string[] = [];

    while (low <= high) {
      const mid = Math.floor((low + high) / 2);
      ctx.font = this.buildFontString(mid, style);

      const lines = this.breakTextIntoLines(
        ctx,
        content,
        maxWidth,
        breakMode,
        config.wordBreak || false
      );

      const totalHeight = lines.length * mid * lineHeight;

      if (totalHeight <= maxHeight) {
        bestFontSize = mid;
        bestLines = lines;
        low = mid + 1;
      } else {
        high = mid - 1;
      }
    }

    // Renderizar com melhor fonte
    ctx.font = this.buildFontString(bestFontSize, style);
    this.renderLines(ctx, layer, bestLines, bestFontSize, lineHeight);
  }

  /**
   * Auto-wrap fixed font size
   */
  private static async renderAutoWrapFixed(
    ctx: CanvasContext,
    layer: Layer,
    content: string,
    config: TextboxConfig,
    options: RenderOptions
  ): Promise<void> {
    const style = layer.style || {};
    const maxWidth = layer.size.width;
    const fontSize = style.fontSize || 16;
    const lineHeight = config.autoWrap?.lineHeight || 1.2;
    const breakMode = config.autoWrap?.breakMode || 'word';

    ctx.font = this.buildFontString(fontSize, style);

    const lines = this.breakTextIntoLines(
      ctx,
      content,
      maxWidth,
      breakMode,
      config.wordBreak || false
    );

    this.renderLines(ctx, layer, lines, fontSize, lineHeight);
  }

  /**
   * Quebra texto em linhas
   */
  private static breakTextIntoLines(
    ctx: CanvasContext,
    text: string,
    maxWidth: number,
    breakMode: 'word' | 'char' | 'hybrid',
    wordBreak: boolean
  ): string[] {
    const lines: string[] = [];

    // Split por newlines explícitos primeiro
    const paragraphs = text.split('\n');

    for (const paragraph of paragraphs) {
      if (breakMode === 'word') {
        // Quebra por palavra
        const words = paragraph.split(' ');
        let currentLine = '';

        for (const word of words) {
          const testLine = currentLine ? `${currentLine} ${word}` : word;
          const metrics = ctx.measureText(testLine);

          if (metrics.width > maxWidth && currentLine) {
            lines.push(currentLine);
            currentLine = word;
          } else {
            currentLine = testLine;
          }
        }

        if (currentLine) {
          lines.push(currentLine);
        }
      } else if (breakMode === 'char') {
        // Quebra por caractere
        let currentLine = '';

        for (const char of paragraph) {
          const testLine = currentLine + char;
          const metrics = ctx.measureText(testLine);

          if (metrics.width > maxWidth && currentLine) {
            lines.push(currentLine);
            currentLine = char;
          } else {
            currentLine = testLine;
          }
        }

        if (currentLine) {
          lines.push(currentLine);
        }
      } else {
        // Hybrid: tenta palavra, fallback para char
        const words = paragraph.split(' ');
        let currentLine = '';

        for (const word of words) {
          const testLine = currentLine ? `${currentLine} ${word}` : word;
          const metrics = ctx.measureText(testLine);

          if (metrics.width > maxWidth && currentLine) {
            lines.push(currentLine);
            currentLine = word;

            // Se palavra sozinha não cabe, quebrar por char
            if (ctx.measureText(word).width > maxWidth) {
              const charLines = this.breakTextIntoLines(ctx, word, maxWidth, 'char', false);
              lines.push(...charLines.slice(0, -1));
              currentLine = charLines[charLines.length - 1];
            }
          } else {
            currentLine = testLine;
          }
        }

        if (currentLine) {
          lines.push(currentLine);
        }
      }
    }

    return lines;
  }

  /**
   * Renderiza múltiplas linhas
   */
  private static renderLines(
    ctx: CanvasContext,
    layer: Layer,
    lines: string[],
    fontSize: number,
    lineHeight: number
  ): void {
    const scaledLineHeight = fontSize * lineHeight;
    const totalHeight = lines.length * scaledLineHeight;

    // Calcular Y inicial baseado no anchor
    const anchor = layer.textboxConfig?.anchor || 'top';
    let startY: number;

    switch (anchor) {
      case 'top':
        startY = layer.position.y + fontSize;
        break;
      case 'middle':
        startY = layer.position.y + (layer.size.height / 2) - (totalHeight / 2) + fontSize;
        break;
      case 'bottom':
        startY = layer.position.y + layer.size.height - totalHeight + fontSize;
        break;
    }

    // Renderizar cada linha
    lines.forEach((line, index) => {
      const x = this.getTextX(layer, ctx.textAlign as string);
      const y = startY + (index * scaledLineHeight);
      ctx.fillText(line, x, y);
    });
  }

  /**
   * Calcula X do texto baseado no alinhamento
   */
  private static getTextX(layer: Layer, align: string): number {
    switch (align) {
      case 'left':
        return layer.position.x;
      case 'center':
        return layer.position.x + layer.size.width / 2;
      case 'right':
        return layer.position.x + layer.size.width;
      default:
        return layer.position.x;
    }
  }

  /**
   * Constrói string de fonte CSS
   */
  private static buildFontString(
    fontSize: number,
    style: LayerStyle
  ): string {
    const fontStyle = style.fontStyle || 'normal';
    const fontWeight = style.fontWeight || 'normal';
    const fontFamily = style.fontFamily || 'sans-serif';

    return `${fontStyle} ${fontWeight} ${fontSize}px ${fontFamily}`;
  }

  /**
   * Renderiza imagem
   */
  private static async renderImage(
    ctx: CanvasContext,
    layer: Layer,
    options: RenderOptions
  ): Promise<void> {
    const { imageLoader, imageCache } = options;

    if (!layer.fileUrl || !imageLoader) return;

    // Carregar imagem (com cache)
    let image: any;

    if (imageCache?.has(layer.fileUrl)) {
      image = imageCache.get(layer.fileUrl);
    } else {
      try {
        image = await imageLoader(layer.fileUrl);
        imageCache?.set(layer.fileUrl, image);
      } catch (error) {
        console.error('Failed to load image:', layer.fileUrl, error);
        return;
      }
    }

    const style = layer.style || {};
    const objectFit = style.objectFit || 'cover';

    const x = layer.position.x;
    const y = layer.position.y;
    const width = layer.size.width;
    const height = layer.size.height;

    // Aplicar opacidade
    if (style.opacity !== undefined) {
      ctx.globalAlpha = style.opacity;
    }

    // Calcular dimensões baseado no objectFit
    const { sx, sy, sw, sh, dx, dy, dw, dh } = this.calculateImageDimensions(
      image.width,
      image.height,
      x,
      y,
      width,
      height,
      objectFit
    );

    // Desenhar imagem
    ctx.drawImage(image, sx, sy, sw, sh, dx, dy, dw, dh);

    // Resetar opacidade
    ctx.globalAlpha = 1.0;
  }

  /**
   * Calcula dimensões da imagem baseado no objectFit
   */
  private static calculateImageDimensions(
    imgWidth: number,
    imgHeight: number,
    x: number,
    y: number,
    width: number,
    height: number,
    objectFit: string
  ) {
    switch (objectFit) {
      case 'cover': {
        const scale = Math.max(width / imgWidth, height / imgHeight);
        const sw = width / scale;
        const sh = height / scale;
        const sx = (imgWidth - sw) / 2;
        const sy = (imgHeight - sh) / 2;

        return { sx, sy, sw, sh, dx: x, dy: y, dw: width, dh: height };
      }

      case 'contain': {
        const scale = Math.min(width / imgWidth, height / imgHeight);
        const dw = imgWidth * scale;
        const dh = imgHeight * scale;
        const dx = x + (width - dw) / 2;
        const dy = y + (height - dh) / 2;

        return { sx: 0, sy: 0, sw: imgWidth, sh: imgHeight, dx, dy, dw, dh };
      }

      case 'fill':
      default:
        return {
          sx: 0,
          sy: 0,
          sw: imgWidth,
          sh: imgHeight,
          dx: x,
          dy: y,
          dw: width,
          dh: height
        };
    }
  }

  /**
   * Renderiza gradiente
   */
  private static async renderGradient(
    ctx: CanvasContext,
    layer: Layer,
    options: RenderOptions
  ): Promise<void> {
    const style = layer.style || {};
    const gradientType = style.gradientType || 'linear';
    const gradientAngle = style.gradientAngle || 0;
    const gradientStops = style.gradientStops || [
      { color: '#FFFFFF', position: 0 },
      { color: '#000000', position: 100 }
    ];

    const x = layer.position.x;
    const y = layer.position.y;
    const width = layer.size.width;
    const height = layer.size.height;

    let gradient: CanvasGradient;

    if (gradientType === 'linear') {
      // Calcular pontos baseado no ângulo
      const angleRad = (gradientAngle * Math.PI) / 180;
      const x1 = x + width / 2 - (Math.cos(angleRad) * width) / 2;
      const y1 = y + height / 2 - (Math.sin(angleRad) * height) / 2;
      const x2 = x + width / 2 + (Math.cos(angleRad) * width) / 2;
      const y2 = y + height / 2 + (Math.sin(angleRad) * height) / 2;

      gradient = ctx.createLinearGradient(x1, y1, x2, y2);
    } else {
      // Radial
      const centerX = x + width / 2;
      const centerY = y + height / 2;
      const radius = Math.max(width, height) / 2;

      gradient = ctx.createRadialGradient(centerX, centerY, 0, centerX, centerY, radius);
    }

    // Adicionar stops
    gradientStops.forEach(stop => {
      gradient.addColorStop(stop.position / 100, stop.color);
    });

    // Desenhar retângulo com gradiente
    ctx.fillStyle = gradient;
    ctx.fillRect(x, y, width, height);
  }

  /**
   * Renderiza logo (reutiliza renderImage)
   */
  private static async renderLogo(
    ctx: CanvasContext,
    layer: Layer,
    options: RenderOptions
  ): Promise<void> {
    return this.renderImage(ctx, layer, options);
  }

  /**
   * Renderiza elemento (reutiliza renderImage)
   */
  private static async renderElement(
    ctx: CanvasContext,
    layer: Layer,
    options: RenderOptions
  ): Promise<void> {
    return this.renderImage(ctx, layer, options);
  }
}
```

### Implementação Backend (Node.js)

**Arquivo:** `src/lib/canvas-renderer.ts`

```typescript
// src/lib/canvas-renderer.ts

import { createCanvas, loadImage, GlobalFonts, SKRSContext2D, Image } from '@napi-rs/canvas';
import { RenderEngine } from './render-engine';
import type { Template, Layer } from '@/types/template';
import { FONT_CONFIG } from './font-config';

export class CanvasRenderer {
  private canvas: any;
  private ctx: SKRSContext2D;
  private width: number;
  private height: number;
  private imageCache: Map<string, Image>;

  constructor(width: number, height: number) {
    this.width = width;
    this.height = height;
    this.canvas = createCanvas(width, height);
    this.ctx = this.canvas.getContext('2d');
    this.imageCache = new Map();

    // Registrar fontes padrão
    this.registerDefaultFonts();
  }

  /**
   * Registra fontes padrão do sistema
   */
  private registerDefaultFonts(): void {
    const fontPaths = FONT_CONFIG.getSystemFontPaths();

    for (const [name, path] of Object.entries(fontPaths)) {
      try {
        GlobalFonts.registerFromPath(path, name);
      } catch (error) {
        console.warn(`Failed to register font ${name}:`, error);
      }
    }
  }

  /**
   * Renderiza template completo
   */
  async renderTemplate(
    template: Template,
    fieldValues: Record<string, any> = {}
  ): Promise<Buffer> {
    // Limpar canvas
    this.ctx.fillStyle = '#FFFFFF';
    this.ctx.fillRect(0, 0, this.width, this.height);

    // Obter layers do designData
    const designData = template.designData as any;
    const layers = designData?.layers || [];

    // Ordenar layers por order (z-index)
    const sortedLayers = [...layers].sort((a, b) => (a.order || 0) - (b.order || 0));

    // Renderizar cada layer
    for (const layer of sortedLayers) {
      await RenderEngine.renderLayer(
        this.ctx,
        layer,
        fieldValues,
        {
          scaleFactor: 1,
          imageLoader: this.nodeImageLoader.bind(this),
          imageCache: this.imageCache,
          fontChecker: this.nodeFontChecker.bind(this),
        }
      );
    }

    // Retornar buffer PNG
    return this.canvas.toBuffer('image/png');
  }

  /**
   * Image loader para Node.js
   */
  private async nodeImageLoader(url: string): Promise<Image> {
    try {
      // Suporta URLs http/https, paths locais e Google Drive
      if (url.startsWith('http://') || url.startsWith('https://')) {
        return await loadImage(url);
      } else if (url.startsWith('/')) {
        // Path local
        return await loadImage(url);
      } else if (url.includes('drive.google.com')) {
        // Google Drive: converter para URL de download direto
        const directUrl = this.convertGoogleDriveUrl(url);
        return await loadImage(directUrl);
      } else {
        throw new Error(`Unsupported image URL: ${url}`);
      }
    } catch (error) {
      console.error('Failed to load image:', url, error);
      throw error;
    }
  }

  /**
   * Converte URL do Google Drive para download direto
   */
  private convertGoogleDriveUrl(url: string): string {
    const fileIdMatch = url.match(/\/d\/([^\/]+)/);
    if (fileIdMatch) {
      const fileId = fileIdMatch[1];
      return `https://drive.google.com/uc?export=download&id=${fileId}`;
    }
    return url;
  }

  /**
   * Font checker para Node.js
   */
  private async nodeFontChecker(fontName: string): Promise<any> {
    const registeredFonts = GlobalFonts.families;
    const isValid = registeredFonts.some(family =>
      family.family.toLowerCase() === fontName.toLowerCase()
    );

    if (!isValid) {
      // Tentar fallback
      const fallback = FONT_CONFIG.getFontWithFallback(fontName);
      return {
        isValid: false,
        fallbackUsed: true,
        fallbackFont: fallback,
        confidence: 0.5,
      };
    }

    return {
      isValid: true,
      fallbackUsed: false,
      confidence: 1.0,
    };
  }
}

/**
 * Helper function para gerar criativo
 */
export async function renderGeneration(generation: any): Promise<string> {
  const template = generation.template;
  const [width, height] = template.dimensions.split('x').map(Number);

  // Criar renderer
  const renderer = new CanvasRenderer(width, height);

  // Renderizar
  const buffer = await renderer.renderTemplate(template, generation.fieldValues);

  const key = `generations/${generation.id}.png`;
  const token = process.env.BLOB_READ_WRITE_TOKEN;
  const result = await put(key, buffer, {
    access: 'public',
    token,
    contentType: 'image/png',
  });

  return result.url;
}

/**
 * Helper function para gerar thumbnail
 */
export async function generateThumbnail(template: Template): Promise<string> {
  // Thumbnail sempre 800x1000 (4:5 ratio)
  const renderer = new CanvasRenderer(800, 1000);

  // Renderizar template vazio (sem fieldValues)
  const buffer = await renderer.renderTemplate(template, {});

  const key = `thumbnails/template-${template.id}.png`;
  const token = process.env.BLOB_READ_WRITE_TOKEN;
  const result = await put(key, buffer, {
    access: 'public',
    token,
    contentType: 'image/png',
  });

  return result.url;
}
```

### Implementação Frontend (Browser)

**Arquivo:** `src/components/editor/CanvasPreview.tsx`

```typescript
// src/components/editor/CanvasPreview.tsx
'use client';

import { useEffect, useRef, useState } from 'react';
import { RenderEngine } from '@/lib/render-engine';
import type { Layer } from '@/types/template';

interface CanvasPreviewProps {
  layers: Layer[];
  fieldValues: Record<string, any>;
  dimensions: string; // "1080x1920"
  className?: string;
  onReady?: (dataUrl: string) => void;
}

export function CanvasPreview({
  layers,
  fieldValues,
  dimensions,
  className,
  onReady,
}: CanvasPreviewProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [imageCache] = useState(new Map<string, HTMLImageElement>());
  const [isRendering, setIsRendering] = useState(false);

  useEffect(() => {
    renderCanvas();
  }, [layers, fieldValues, dimensions]);

  async function renderCanvas() {
    if (!canvasRef.current || isRendering) return;

    setIsRendering(true);

    try {
      const canvas = canvasRef.current;
      const ctx = canvas.getContext('2d');
      if (!ctx) return;

      const [width, height] = dimensions.split('x').map(Number);

      // Ajustar tamanho do canvas
      canvas.width = width;
      canvas.height = height;

      // Limpar canvas
      ctx.fillStyle = '#FFFFFF';
      ctx.fillRect(0, 0, width, height);

      // Ordenar layers por order
      const sortedLayers = [...layers].sort((a, b) => (a.order || 0) - (b.order || 0));

      // Renderizar cada layer
      for (const layer of sortedLayers) {
        await RenderEngine.renderLayer(
          ctx,
          layer,
          fieldValues,
          {
            scaleFactor: 1,
            imageLoader: browserImageLoader,
            imageCache,
            fontChecker: browserFontChecker,
          }
        );
      }

      // Callback com data URL
      if (onReady) {
        const dataUrl = canvas.toDataURL('image/png');
        onReady(dataUrl);
      }
    } catch (error) {
      console.error('Failed to render canvas:', error);
    } finally {
      setIsRendering(false);
    }
  }

  return (
    <canvas
      ref={canvasRef}
      className={className}
      style={{
        width: '100%',
        height: '100%',
        objectFit: 'contain',
      }}
    />
  );
}

/**
 * Image loader para browser
 */
async function browserImageLoader(url: string): Promise<HTMLImageElement> {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.crossOrigin = 'anonymous';

    img.onload = () => resolve(img);
    img.onerror = () => reject(new Error(`Failed to load image: ${url}`));

    img.src = url;
  });
}

/**
 * Font checker para browser
 */
async function browserFontChecker(fontName: string): Promise<any> {
  await document.fonts.ready;

  const isValid = document.fonts.check(`16px ${fontName}`);

  if (!isValid) {
    // Fallback para fonte segura
    const fallback = fontName.includes(',')
      ? fontName.split(',')[1].trim()
      : 'sans-serif';

    return {
      isValid: false,
      fallbackUsed: true,
      fallbackFont: fallback,
      confidence: 0.5,
    };
  }

  return {
    isValid: true,
    fallbackUsed: false,
    confidence: 1.0,
  };
}
```

### Configuração de Fontes

**Arquivo:** `src/lib/font-config.ts`

```typescript
// src/lib/font-config.ts

export const FONT_CONFIG = {
  /**
   * Fontes seguras (sempre disponíveis)
   */
  SAFE_FONTS: [
    'Montserrat',
    'Arial',
    'Helvetica',
    'sans-serif',
    'serif',
    'monospace',
  ],

  /**
   * Fonte padrão
   */
  DEFAULT_FONT: 'Montserrat',

  /**
   * Obtém fonte com fallback
   */
  getFontWithFallback(fontName: string): string {
    const safeFonts = this.SAFE_FONTS;

    // Se já é fonte segura, retornar
    if (safeFonts.includes(fontName)) {
      return fontName;
    }

    // Retornar com fallback
    return `${fontName}, ${this.DEFAULT_FONT}, sans-serif`;
  },

  /**
   * Normaliza string de fonte
   */
  normalizeFontString(
    fontSize: number,
    fontFamily: string,
    fontWeight: string | number = 'normal',
    fontStyle: string = 'normal'
  ): string {
    const family = this.getFontWithFallback(fontFamily);
    return `${fontStyle} ${fontWeight} ${fontSize}px ${family}`;
  },

  /**
   * Obtém paths de fontes do sistema (Node.js)
   */
  getSystemFontPaths(): Record<string, string> {
    const platform = process.platform;

    if (platform === 'darwin') {
      // macOS
      return {
        'Montserrat': '/System/Library/Fonts/Supplemental/Arial.ttf',
        'Arial': '/System/Library/Fonts/Supplemental/Arial.ttf',
        'Helvetica': '/System/Library/Fonts/Helvetica.ttc',
      };
    } else if (platform === 'linux') {
      // Linux
      return {
        'Montserrat': '/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf',
        'Arial': '/usr/share/fonts/truetype/liberation/LiberationSans-Regular.ttf',
      };
    } else if (platform === 'win32') {
      // Windows
      return {
        'Montserrat': 'C:\\Windows\\Fonts\\Arial.ttf',
        'Arial': 'C:\\Windows\\Fonts\\Arial.ttf',
        'Helvetica': 'C:\\Windows\\Fonts\\Arial.ttf',
      };
    }

    return {};
  },
};
```

## Componentes Frontend

### Editor de Templates

**Página:** `src/app/(protected)/templates/[id]/editor/page.tsx`

```typescript
// src/app/(protected)/templates/[id]/editor/page.tsx
'use client';

import { useState, useEffect } from 'react';
import { useParams } from 'next/navigation';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '@/lib/api-client';
import { Canvas } from '@/components/editor/Canvas';
import { LayersPanel } from '@/components/editor/LayersPanel';
import { PropertiesPanel } from '@/components/editor/PropertiesPanel';
import { Toolbar } from '@/components/editor/Toolbar';
import { CanvasPreview } from '@/components/editor/CanvasPreview';
import type { Template, Layer } from '@/types/template';

export default function TemplateEditorPage() {
  const params = useParams();
  const templateId = parseInt(params.id as string);

  const queryClient = useQueryClient();

  // Estado local
  const [layers, setLayers] = useState<Layer[]>([]);
  const [selectedLayerId, setSelectedLayerId] = useState<string | null>(null);
  const [previewMode, setPreviewMode] = useState(false);

  // Carregar template
  const { data: template, isLoading } = useQuery<Template>({
    queryKey: ['template', templateId],
    queryFn: () => api.get(`/api/templates/${templateId}`),
  });

  // Atualizar layers quando template carrega
  useEffect(() => {
    if (template?.designData) {
      const designData = template.designData as any;
      setLayers(designData.layers || []);
    }
  }, [template]);

  // Mutation para salvar
  const saveMutation = useMutation({
    mutationFn: (data: Partial<Template>) =>
      api.put(`/api/templates/${templateId}`, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['template', templateId] });
    },
  });

  // Handlers
  const handleSave = async () => {
    if (!template) return;

    await saveMutation.mutateAsync({
      designData: {
        ...template.designData,
        layers,
      },
    });
  };

  const handleLayerUpdate = (layerId: string, updates: Partial<Layer>) => {
    setLayers(prev =>
      prev.map(layer =>
        layer.id === layerId ? { ...layer, ...updates } : layer
      )
    );
  };

  const handleAddLayer = (type: Layer['type']) => {
    const newLayer: Layer = {
      id: `layer-${Date.now()}`,
      type,
      name: `${type} Layer`,
      visible: true,
      locked: false,
      order: layers.length,
      position: { x: 100, y: 100 },
      size: { width: 200, height: 100 },
      content: type === 'text' ? 'Texto' : undefined,
      style: {},
    };

    setLayers(prev => [...prev, newLayer]);
    setSelectedLayerId(newLayer.id);
  };

  const handleDeleteLayer = (layerId: string) => {
    setLayers(prev => prev.filter(layer => layer.id !== layerId));
    if (selectedLayerId === layerId) {
      setSelectedLayerId(null);
    }
  };

  const selectedLayer = layers.find(l => l.id === selectedLayerId);

  if (isLoading) {
    return <div>Carregando...</div>;
  }

  if (!template) {
    return <div>Template não encontrado</div>;
  }

  return (
    <div className="flex h-screen">
      {/* Layers Panel (Left) */}
      <div className="w-64 border-r">
        <LayersPanel
          layers={layers}
          selectedLayerId={selectedLayerId}
          onSelectLayer={setSelectedLayerId}
          onUpdateLayer={handleLayerUpdate}
          onDeleteLayer={handleDeleteLayer}
          onReorder={setLayers}
        />
      </div>

      {/* Canvas Area (Center) */}
      <div className="flex-1 flex flex-col">
        <Toolbar
          onAddLayer={handleAddLayer}
          onSave={handleSave}
          isSaving={saveMutation.isPending}
          previewMode={previewMode}
          onTogglePreview={() => setPreviewMode(!previewMode)}
        />

        <div className="flex-1 bg-gray-100 p-8 overflow-auto">
          {previewMode ? (
            <CanvasPreview
              layers={layers}
              fieldValues={{}}
              dimensions={template.dimensions}
              className="mx-auto shadow-lg"
            />
          ) : (
            <Canvas
              layers={layers}
              selectedLayerId={selectedLayerId}
              onSelectLayer={setSelectedLayerId}
              onUpdateLayer={handleLayerUpdate}
              dimensions={template.dimensions}
            />
          )}
        </div>
      </div>

      {/* Properties Panel (Right) */}
      {selectedLayer && (
        <div className="w-80 border-l">
          <PropertiesPanel
            layer={selectedLayer}
            onUpdate={(updates) => handleLayerUpdate(selectedLayer.id, updates)}
          />
        </div>
      )}
    </div>
  );
}
```

### Studio de Geração

**Página:** `src/app/(protected)/projects/[id]/studio/page.tsx`

```typescript
// src/app/(protected)/projects/[id]/studio/page.tsx
'use client';

import { useState } from 'react';
import { useParams } from 'next/navigation';
import { useQuery, useMutation } from '@tanstack/react-query';
import { api } from '@/lib/api-client';
import { TemplateSelector } from '@/components/studio/TemplateSelector';
import { PhotoSelector } from '@/components/studio/PhotoSelector';
import { StudioCanvas } from '@/components/studio/StudioCanvas';
import { FieldsForm } from '@/components/studio/FieldsForm';
import { Button } from '@/components/ui/button';
import type { Template, Layer } from '@/types/template';

export default function StudioPage() {
  const params = useParams();
  const projectId = parseInt(params.id as string);

  const [selectedTemplateId, setSelectedTemplateId] = useState<number | null>(null);
  const [fieldValues, setFieldValues] = useState<Record<string, any>>({});
  const [step, setStep] = useState<'template' | 'photo' | 'edit'>('template');

  // Carregar templates do projeto
  const { data: templates } = useQuery({
    queryKey: ['templates', projectId],
    queryFn: () => api.get(`/api/projects/${projectId}/templates`),
  });

  // Carregar template selecionado
  const { data: selectedTemplate } = useQuery({
    queryKey: ['template', selectedTemplateId],
    queryFn: () => api.get(`/api/templates/${selectedTemplateId}`),
    enabled: !!selectedTemplateId,
  });

  // Mutation para gerar criativo
  const generateMutation = useMutation({
    mutationFn: (data: any) =>
      api.post(`/api/projects/${projectId}/generations`, data),
    onSuccess: (result) => {
      // Exibir resultado
      console.log('Criativo gerado:', result);
    },
  });

  const handleSelectTemplate = (templateId: number) => {
    setSelectedTemplateId(templateId);
    setStep('photo');
  };

  const handleSelectPhoto = (photoUrl: string) => {
    // Identificar layer de imagem dinâmica
    const designData = selectedTemplate?.designData as any;
    const imageLayer = designData?.layers?.find(
      (l: Layer) => l.type === 'image' && l.isDynamic
    );

    if (imageLayer) {
      setFieldValues(prev => ({ ...prev, [imageLayer.id]: photoUrl }));
    }

    setStep('edit');
  };

  const handleGenerate = async () => {
    if (!selectedTemplateId) return;

    await generateMutation.mutateAsync({
      templateId: selectedTemplateId,
      fieldValues,
    });
  };

  return (
    <div className="container mx-auto p-8">
      {step === 'template' && (
        <div>
          <h1 className="text-2xl font-bold mb-4">Escolha um Template</h1>
          <TemplateSelector
            templates={templates || []}
            onSelect={handleSelectTemplate}
          />
        </div>
      )}

      {step === 'photo' && selectedTemplate && (
        <div>
          <h1 className="text-2xl font-bold mb-4">Selecione uma Foto</h1>
          <PhotoSelector
            projectId={projectId}
            onSelect={handleSelectPhoto}
            onSkip={() => setStep('edit')}
          />
        </div>
      )}

      {step === 'edit' && selectedTemplate && (
        <div className="grid grid-cols-2 gap-8">
          <div>
            <h2 className="text-xl font-bold mb-4">Preview</h2>
            <StudioCanvas
              template={selectedTemplate}
              fieldValues={fieldValues}
            />
          </div>

          <div>
            <h2 className="text-xl font-bold mb-4">Campos Dinâmicos</h2>
            <FieldsForm
              template={selectedTemplate}
              values={fieldValues}
              onChange={setFieldValues}
            />

            <Button
              onClick={handleGenerate}
              disabled={generateMutation.isPending}
              className="mt-4 w-full"
            >
              {generateMutation.isPending ? 'Gerando...' : 'Gerar Criativo'}
            </Button>
          </div>
        </div>
      )}
    </div>
  );
}
```

## Schemas de Validação (Zod)

```typescript
// src/lib/validations/template.ts

import { z } from 'zod';

export const layerSchema = z.object({
  id: z.string(),
  type: z.enum(['text', 'image', 'gradient', 'gradient2', 'logo', 'element']),
  name: z.string(),
  visible: z.boolean(),
  locked: z.boolean(),
  order: z.number(),
  position: z.object({ x: z.number(), y: z.number() }),
  size: z.object({ width: z.number(), height: z.number() }),
  rotation: z.number().optional(),
  content: z.string().optional(),
  style: z.record(z.any()).optional(),
  isDynamic: z.boolean().optional(),
  textboxConfig: z.object({
    spacing: z.number().optional(),
    anchor: z.enum(['top', 'middle', 'bottom']).optional(),
    textMode: z.enum(['auto-resize-single', 'auto-resize-multi', 'auto-wrap-fixed', 'fitty']).optional(),
    autoResize: z.object({
      minFontSize: z.number(),
      maxFontSize: z.number(),
    }).optional(),
    autoWrap: z.object({
      lineHeight: z.number(),
      breakMode: z.enum(['word', 'char', 'hybrid']),
      autoExpand: z.boolean(),
    }).optional(),
    wordBreak: z.boolean().optional(),
  }).optional(),
  logoId: z.number().optional(),
  elementId: z.number().optional(),
  fileUrl: z.string().optional(),
  parentId: z.string().nullable().optional(),
});

export const designDataSchema = z.object({
  canvas: z.object({
    width: z.number(),
    height: z.number(),
    backgroundColor: z.string(),
  }),
  layers: z.array(layerSchema),
});

export const createTemplateSchema = z.object({
  name: z.string().min(1, 'Nome é obrigatório'),
  type: z.enum(['STORY', 'FEED', 'SQUARE']),
  dimensions: z.string().regex(/^\d+x\d+$/, 'Formato inválido'),
});

export const updateTemplateSchema = z.object({
  name: z.string().min(1).optional(),
  designData: designDataSchema.optional(),
  dynamicFields: z.array(z.any()).optional(),
});

export const createGenerationSchema = z.object({
  templateId: z.number(),
  fieldValues: z.record(z.any()),
});

export const createCarouselSchema = z.object({
  templateId: z.number(),
  slides: z.array(z.object({
    fieldValues: z.record(z.any()),
  })).min(2).max(10),
});
```

## Fluxo de Desenvolvimento

### Fase 1: Setup e Fundação
1. ✅ Criar schema Prisma e migrations
2. ✅ Implementar RenderEngine unificado
3. ✅ Implementar CanvasRenderer (backend)
4. ✅ Configurar Vercel Blob upload
5. ✅ Criar APIs base de Projetos e Templates

### Fase 2: Editor de Templates
1. Criar página do editor (`/templates/[id]/editor`)
2. Implementar Canvas DOM (drag-and-drop)
3. Implementar Layers Panel
4. Implementar Properties Panel
5. Implementar Toolbar com ações
6. Adicionar CanvasPreview (preview unificado)

### Fase 3: Studio de Geração
1. Criar página do studio (`/projects/[id]/studio`)
2. Implementar TemplateSelector
3. Implementar PhotoSelector (upload + Google Drive)
4. Implementar StudioCanvas (preview em tempo real)
5. Implementar FieldsForm (formulário de campos dinâmicos)
6. Integrar geração backend

### Fase 4: Gestão de Criativos
1. Criar página de listagem (`/projects/[id]/creativos`)
2. Implementar grid/list view
3. Implementar filtros e busca
4. Implementar download individual/bulk
5. Implementar lightbox (PhotoSwipe)

### Fase 5: Assets e Integrações
1. Implementar upload de logos
2. Implementar upload de elementos
3. Implementar upload de fontes customizadas
4. Integrar Google Drive API (opcional)

### Fase 6: Testes e Refinamentos
1. Testar consistência frontend ↔ backend
2. Comparar preview com geração final
3. Testar carrossel (múltiplos slides)
4. Otimizar performance (caching, lazy loading)
5. Testes de acessibilidade

## Checklists

### Backend Development
- [ ] Criar schema Prisma completo
- [ ] Implementar RenderEngine.ts (compartilhado)
- [ ] Implementar CanvasRenderer.ts (Node.js)
- [ ] Criar API routes com Zod validation
- [ ] Implementar auth checks (Clerk)
- [ ] Implementar upload para Vercel Blob
- [ ] Testar renderização de texto (quebra de linha)
- [ ] Testar renderização de gradientes
- [ ] Testar renderização de imagens (objectFit)
- [ ] Implementar cache de imagens
- [ ] Implementar sistema de fontes

### Frontend Development
- [ ] Criar CanvasPreview.tsx (preview unificado)
- [ ] Criar Canvas.tsx (editor DOM)
- [ ] Criar LayersPanel.tsx
- [ ] Criar PropertiesPanel.tsx
- [ ] Criar Toolbar.tsx
- [ ] Criar TemplateSelector.tsx
- [ ] Criar PhotoSelector.tsx
- [ ] Criar StudioCanvas.tsx
- [ ] Criar FieldsForm.tsx
- [ ] Implementar TanStack Query hooks
- [ ] Implementar upload de arquivos
- [ ] Implementar drag-and-drop
- [ ] Testar preview em tempo real
- [ ] Testar responsividade

### Database Development
- [ ] Criar migration inicial
- [ ] Adicionar índices apropriados
- [ ] Testar cascata de deletes
- [ ] Testar relações (Project → Template → Generation)
- [ ] Implementar soft delete (se necessário)
- [ ] Testar performance de queries

### Security Check
- [ ] Validar auth em todas as rotas
- [ ] Verificar ownership de recursos
- [ ] Sanitizar HTML/MD em conteúdo
- [ ] Validar uploads (tipo, tamanho)
- [ ] Implementar rate limiting
- [ ] Logs sem dados sensíveis
- [ ] Testar isolamento multi-tenant

## Observabilidade e Monitoramento

### Logs Mínimos
```typescript
// Exemplo de log seguro
logger.info('Generation started', {
  generationId: generation.id,
  templateId: generation.templateId,
  userId: generation.createdBy,
  // NÃO logar fieldValues (pode ter dados sensíveis)
});

logger.info('Generation completed', {
  generationId: generation.id,
  duration: Date.now() - startTime,
  resultUrl: generation.resultUrl,
});
```

### Métricas Básicas
- Tempo de renderização (avg, p95, p99)
- Taxa de erro de geração
- Uso de cache de imagens (hit rate)
- Uploads por minuto
- Templates criados por dia

## Entregáveis do PR

### Código
- [ ] Schema Prisma + migration
- [ ] `src/lib/render-engine.ts` (compartilhado)
- [ ] `src/lib/canvas-renderer.ts` (backend)
- [ ] `src/lib/font-config.ts` (configuração de fontes)
- [ ] API routes (projetos, templates, generations)
- [ ] Páginas frontend (editor, studio, listagem)
- [ ] Componentes (Canvas, CanvasPreview, LayersPanel, etc)
- [ ] Custom hooks (TanStack Query)
- [ ] Schemas de validação (Zod)

### Documentação
- [ ] README atualizado com setup
- [ ] `.env.example` atualizado
- [ ] Comentários em código complexo (RenderEngine)
- [ ] Documentação de APIs (opcional: OpenAPI)

### Qualidade
- [ ] `npm run lint` sem erros
- [ ] `npm run typecheck` sem erros
- [ ] `npm run build` sem erros
- [ ] Testes manuais de consistência (preview = geração)
- [ ] Testes de multi-tenant (isolamento)

## Notas Finais

### Diferenças do V1 para V2

| Aspecto | Studio Lagosta V1 | Studio Lagosta V2 |
|---------|-------------------|-------------------|
| Framework | React + Express (separados) | Next.js 15 (unificado) |
| Auth | Replit Auth | Clerk |
| Database | PostgreSQL + Drizzle | PostgreSQL + Prisma |
| Storage | Local + Vercel Blob | Vercel Blob apenas |
| Styling | Tailwind v3 | Tailwind v4 |
| Forms | React Hook Form | React Hook Form + Zod |
| State | Zustand | TanStack Query (server state) |
| Preview | DOM apenas | **DOM + Canvas (unificado)** |

### Vantagens da Nova Arquitetura

1. **Renderização Unificada**: Preview = Geração Final (100%)
2. **Performance**: TanStack Query com cache automático
3. **Type Safety**: Prisma + TypeScript + Zod end-to-end
4. **Autenticação**: Clerk com middleware automático
5. **Deployment**: Next.js com otimizações automáticas
6. **Manutenção**: Codebase unificado (não mais frontend/backend separados)

### Próximos Passos (Fase 2)

- Publicação no Instagram/Facebook
- Templates marketplace
- IA para geração de textos (Vercel AI SDK)
- Analytics avançados
- A/B testing de criativos
- Mobile app (React Native com mesmo RenderEngine)

---

**Pronto para começar!** 🦞✨

Este prompt fornece todas as informações necessárias para implementar o sistema completo de templates e criativos no Studio Lagosta V2, com renderização unificada usando HTML5 Canvas tanto no frontend quanto no backend, eliminando as inconsistências do V1.
