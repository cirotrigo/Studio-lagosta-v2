# Plano de Implementação: Editor Profissional com Konva.js (React-Konva)

**Projeto:** Studio Lagosta - Template Editor v2.0 (Canva-like)
**Data:** Outubro 2025
**Objetivo:** Transformar o editor atual em uma solução profissional tipo Canva usando Konva.js, com sistema completo de templates, ferramentas de edição de imagem, asset library e interface polida

---

## 📋 Sumário Executivo

Este plano detalha a transformação completa do editor de templates do Studio Lagosta em uma plataforma profissional de design visual, inspirada no Canva, utilizando **Konva.js (via react-konva)** como engine de renderização, com funcionalidades avançadas de edição, biblioteca de assets e sistema de templates.

### Motivação

**Problemas Atuais:**
- Editor básico sem ferramentas de edição avançadas
- Falta de biblioteca de assets integrada
- Interface não intuitiva para usuários não-técnicos
- Ausência de templates prontos para uso
- Renderização limitada sem preview em tempo real interativo

**Nova Visão - Editor Tipo Canva:**
- ✅ **Interface polida e intuitiva** (drag & drop natural)
- ✅ **Biblioteca de templates** (categorias, preview, one-click load)
- ✅ **Asset library completa** (imagens, ícones, formas, backgrounds)
- ✅ **Ferramentas de edição de imagem** (crop, rotate, filters, brightness, contrast)
- ✅ **Preview em tempo real** (WYSIWYG total)
- ✅ **Performance profissional** (canvas nativo, 60 FPS)
- ✅ **Zero custo de licenciamento** (MIT License)

---

## 🎨 Visão do Produto Final

### Interface Principal (Canva-like Layout)

```
┌─────────────────────────────────────────────────────────────────────┐
│  [Logo] Studio Lagosta Editor        [Save] [Export] [Share] [User] │
├──────────┬──────────────────────────────────────────────┬───────────┤
│          │                                              │           │
│  SIDEBAR │              CANVAS AREA                     │ PROPERTIES│
│          │                                              │   PANEL   │
│  - Templates                                            │           │
│  - Text  │         ┌─────────────────────┐            │  Selected:│
│  - Images│         │                     │            │  Text     │
│  - Shapes│         │   [Active Canvas]   │            │           │
│  - Icons │         │                     │            │  Font:    │
│  - Upload│         │   Zoom: 100%        │            │  [Inter]  │
│  - BG    │         └─────────────────────┘            │           │
│          │                                              │  Size:    │
│          │  [Layers Panel - Minimized]                 │  [24px]   │
│          │                                              │           │
│  260px   │                                              │  300px    │
└──────────┴──────────────────────────────────────────────┴───────────┘
```

### Funcionalidades Core

#### 1. **Sistema de Templates**
- Biblioteca categorizada (Social Media, Business Cards, Flyers, Posters)
- Preview em thumbnails
- One-click para carregar template
- Salvar templates customizados
- Template marketplace (futuro)

#### 2. **Asset Library**
- **Imagens**: Unsplash integration, Google Drive, upload local
- **Ícones**: Font Awesome, Material Icons, custom SVGs
- **Formas**: Retângulos, círculos, polígonos, estrelas, setas
- **Backgrounds**: Gradientes, patterns, texturas
- **Elementos**: Linhas, divisores, molduras

#### 3. **Ferramentas de Edição de Imagem**
- **Transform**: Crop, rotate, flip (H/V), resize
- **Filters**: Blur, sharpen, brightness, contrast, saturation, sepia, grayscale, invert
- **Adjustments**: Opacity, blend modes
- **Effects**: Shadow, glow, border
- **Background Removal**: Integração com API (opcional)

#### 4. **Interface Canva-like**
- **Toolbar superior**: Ações principais (save, export, undo/redo)
- **Sidebar esquerdo**: Categorias de assets
- **Canvas central**: Editor interativo com zoom/pan
- **Properties panel direito**: Propriedades do elemento selecionado
- **Layers panel inferior**: Gestão de camadas (collapsible)

#### 5. **Preview em Tempo Real**
- Canvas é WYSIWYG (what you see is what you get)
- Renderização instantânea de mudanças
- Export preview antes de gerar arquivo final

---

## 🏗️ Arquitetura Técnica

### Por Que Konva.js em Vez de Fabric.js?

**Vantagens do Konva.js:**
1. ✅ **React-first**: `react-konva` é wrapper oficial com componentes React nativos
2. ✅ **Performance superior**: Mais rápido para operações complexas
3. ✅ **API moderna**: Mais intuitivo e menos boilerplate
4. ✅ **Menor curva de aprendizado**: Component-based vs imperative
5. ✅ **Melhor TypeScript support**: Types nativos e atualizados
6. ✅ **Usado por grandes players**: Meta, Microsoft, Polotno SDK
7. ✅ **Menos issues abertas**: 56 vs 241 (Fabric.js)

**Nota sobre Next.js 15:**
- Issue aberta sobre Next.js 15 (Jan 2025)
- **Solução**: Dynamic import com `ssr: false` (padrão comprovado)

### Stack Tecnológica

```json
{
  "dependencies": {
    "konva": "^9.3.0",
    "react-konva": "^19.0.10",
    "react-konva-utils": "^1.0.6",
    "use-image": "^1.1.1",
    "@dnd-kit/core": "^6.1.0",
    "@dnd-kit/sortable": "^8.0.0",
    "unsplash-js": "^7.0.19",
    "react-colorful": "^5.6.1",
    "cropperjs": "^1.6.2"
  }
}
```

### Configuração Next.js 15

**next.config.ts** (atualizar):

```typescript
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  webpack: (config) => {
    config.externals = [...(config.externals || []), { canvas: 'canvas' }]
    return config
  },
  images: {
    remotePatterns: [
      { hostname: 'images.unsplash.com' },
      { hostname: 'source.unsplash.com' },
      { hostname: 'cdn.example.com' }, // Seus assets
    ],
  },
}

export default nextConfig
```

### Estrutura de Arquivos (Nova)

```
src/
├── components/
│   ├── editor/
│   │   ├── konva-editor.tsx              # NOVO: Editor principal
│   │   ├── konva-stage.tsx               # NOVO: Stage wrapper
│   │   ├── konva-layer-renderer.tsx      # NOVO: Layer renderer
│   │   ├── konva-transformer.tsx         # NOVO: Transform controls
│   │   ├── toolbar/
│   │   │   ├── editor-toolbar.tsx        # REFACTOR: Toolbar superior
│   │   │   ├── zoom-controls.tsx         # NOVO: Zoom in/out/fit
│   │   │   ├── alignment-tools.tsx       # NOVO: Align/distribute
│   │   │   └── history-controls.tsx      # NOVO: Undo/redo
│   │   ├── sidebar/
│   │   │   ├── editor-sidebar.tsx        # NOVO: Sidebar principal
│   │   │   ├── templates-panel.tsx       # NOVO: Templates
│   │   │   ├── text-panel.tsx            # NOVO: Text tools
│   │   │   ├── images-panel.tsx          # NOVO: Image assets
│   │   │   ├── shapes-panel.tsx          # NOVO: Shapes library
│   │   │   ├── icons-panel.tsx           # NOVO: Icons library
│   │   │   ├── uploads-panel.tsx         # NOVO: User uploads
│   │   │   └── backgrounds-panel.tsx     # NOVO: Backgrounds
│   │   ├── properties/
│   │   │   ├── properties-panel.tsx      # REFACTOR: Panel direito
│   │   │   ├── text-properties.tsx       # NOVO: Text props
│   │   │   ├── image-properties.tsx      # NOVO: Image props + filters
│   │   │   ├── shape-properties.tsx      # NOVO: Shape props
│   │   │   └── position-properties.tsx   # NOVO: Position/size
│   │   ├── layers/
│   │   │   ├── layers-panel.tsx          # REFACTOR: Layers list
│   │   │   ├── layer-item.tsx            # NOVO: Single layer
│   │   │   └── layer-context-menu.tsx    # NOVO: Right-click menu
│   │   └── modals/
│   │       ├── image-editor-modal.tsx    # NOVO: Advanced image editor
│   │       ├── crop-modal.tsx            # NOVO: Crop tool
│   │       └── export-modal.tsx          # NOVO: Export options
│   └── templates/                         # Componentes antigos (deprecar)
├── lib/
│   ├── konva/
│   │   ├── konva-adapter.ts              # NOVO: Adapter pattern
│   │   ├── konva-layer-factory.tsx       # NOVO: Create Konva nodes
│   │   ├── konva-serializer.ts           # NOVO: DesignData ↔ Konva
│   │   ├── konva-filters.ts              # NOVO: Image filters
│   │   ├── konva-utils.ts                # NOVO: Helpers
│   │   └── konva-export.ts               # NOVO: Export logic
│   ├── assets/
│   │   ├── unsplash-service.ts           # NOVO: Unsplash API
│   │   ├── icons-service.ts              # NOVO: Icons provider
│   │   ├── shapes-library.ts             # NOVO: Shapes definitions
│   │   └── templates-service.ts          # NOVO: Templates CRUD
│   ├── history/
│   │   ├── history-manager.ts            # NOVO: Undo/redo stack
│   │   └── history-hooks.ts              # NOVO: useHistory hook
│   └── render-engine.ts                  # MANTIDO: Backend rendering
├── hooks/
│   ├── editor/
│   │   ├── use-konva-editor.ts           # NOVO: Main editor hook
│   │   ├── use-konva-stage.ts            # NOVO: Stage management
│   │   ├── use-selection.ts              # NOVO: Selection state
│   │   ├── use-history.ts                # NOVO: Undo/redo
│   │   └── use-shortcuts.ts              # NOVO: Keyboard shortcuts
│   ├── use-templates.ts                  # NOVO: Templates queries
│   └── use-assets.ts                     # NOVO: Assets queries
└── types/
    ├── konva.d.ts                        # NOVO: Konva type extensions
    └── assets.ts                         # NOVO: Asset types
```

---

## 🔄 Mapeamento: Layer Types → Konva Nodes

### Text Layer → Konva.Text
```tsx
import { Text } from 'react-konva'
import { useImage } from 'use-image'

// Studio Lagosta Layer
const textLayer: Layer = {
  type: 'text',
  content: 'Hello World',
  style: {
    fontSize: 24,
    fontFamily: 'Inter',
    color: '#000000',
    textAlign: 'center'
  }
}

// Konva Component
<Text
  text={textLayer.content}
  fontSize={textLayer.style.fontSize}
  fontFamily={textLayer.style.fontFamily}
  fill={textLayer.style.color}
  align={textLayer.style.textAlign}
  x={textLayer.position.x}
  y={textLayer.position.y}
  draggable={!textLayer.locked}
  onDragEnd={(e) => updatePosition(e.target.x(), e.target.y())}
/>
```

### Image Layer → Konva.Image
```tsx
import { Image } from 'react-konva'
import useImage from 'use-image'
import Konva from 'konva'

function ImageLayer({ layer }: { layer: Layer }) {
  const [image] = useImage(layer.fileUrl!, 'anonymous')
  const imageRef = React.useRef<Konva.Image>(null)

  // Apply filters
  React.useEffect(() => {
    if (image && imageRef.current) {
      imageRef.current.cache()
      imageRef.current.getLayer()?.batchDraw()
    }
  }, [image, layer.style?.filters])

  return (
    <Image
      ref={imageRef}
      image={image}
      x={layer.position.x}
      y={layer.position.y}
      width={layer.size.width}
      height={layer.size.height}
      rotation={layer.rotation}
      filters={layer.style?.filters?.map(f => Konva.Filters[f])}
      brightness={layer.style?.brightness}
      contrast={layer.style?.contrast}
      draggable={!layer.locked}
    />
  )
}
```

### Gradient Layer → Konva.Rect with fillLinearGradientColorStops
```tsx
import { Rect } from 'react-konva'

const gradientLayer: Layer = {
  type: 'gradient',
  style: {
    gradientType: 'linear',
    gradientAngle: 90,
    gradientStops: [
      { color: '#ff0000', position: 0 },
      { color: '#0000ff', position: 1 }
    ]
  }
}

// Konva Component
<Rect
  x={gradientLayer.position.x}
  y={gradientLayer.position.y}
  width={gradientLayer.size.width}
  height={gradientLayer.size.height}
  fillLinearGradientStartPoint={{ x: 0, y: 0 }}
  fillLinearGradientEndPoint={{ x: 0, y: gradientLayer.size.height }}
  fillLinearGradientColorStops={[
    0, '#ff0000',
    1, '#0000ff'
  ]}
/>
```

### Shape Layer → Konva.Rect / Circle / Star
```tsx
import { Rect, Circle, Star } from 'react-konva'

// Retângulo
<Rect
  x={layer.position.x}
  y={layer.position.y}
  width={layer.size.width}
  height={layer.size.height}
  fill={layer.style?.fill}
  stroke={layer.style?.stroke}
  strokeWidth={layer.style?.strokeWidth}
  cornerRadius={layer.style?.cornerRadius}
/>

// Círculo
<Circle
  x={layer.position.x}
  y={layer.position.y}
  radius={layer.size.width / 2}
  fill={layer.style?.fill}
/>

// Estrela
<Star
  x={layer.position.x}
  y={layer.position.y}
  numPoints={5}
  innerRadius={layer.size.width / 4}
  outerRadius={layer.size.width / 2}
  fill={layer.style?.fill}
/>
```

---

## 🎯 Implementação: Componente Principal

### KonvaEditor.tsx (Editor Completo)

```tsx
"use client"

import * as React from 'react'
import dynamic from 'next/dynamic'
import { Stage, Layer } from 'react-konva'
import { KonvaEventObject } from 'konva/lib/Node'
import { useTemplateEditor } from '@/contexts/template-editor-context'
import { KonvaLayerFactory } from '@/lib/konva/konva-layer-factory'
import { EditorToolbar } from './toolbar/editor-toolbar'
import { EditorSidebar } from './sidebar/editor-sidebar'
import { PropertiesPanel } from './properties/properties-panel'
import { LayersPanel } from './layers/layers-panel'
import { KonvaTransformer } from './konva-transformer'

// Dynamic import para evitar SSR issues
const KonvaEditorCore = dynamic(
  () => Promise.resolve(KonvaEditorCoreComponent),
  { ssr: false }
)

function KonvaEditorCoreComponent() {
  const stageRef = React.useRef<any>(null)
  const layerRef = React.useRef<any>(null)

  const {
    design,
    selectedLayerId,
    selectLayer,
    updateLayer,
    zoom,
    setZoom,
  } = useTemplateEditor()

  const canvasWidth = design.canvas.width
  const canvasHeight = design.canvas.height

  // Handle shape selection
  const handleStageClick = (e: KonvaEventObject<MouseEvent>) => {
    const clickedOnEmpty = e.target === e.target.getStage()
    if (clickedOnEmpty) {
      selectLayer(null)
    }
  }

  // Handle zoom with mouse wheel
  const handleWheel = (e: KonvaEventObject<WheelEvent>) => {
    e.evt.preventDefault()

    const stage = stageRef.current
    const oldScale = stage.scaleX()
    const pointer = stage.getPointerPosition()

    const mousePointTo = {
      x: (pointer.x - stage.x()) / oldScale,
      y: (pointer.y - stage.y()) / oldScale,
    }

    const direction = e.evt.deltaY > 0 ? -1 : 1
    const newScale = Math.max(0.25, Math.min(2, oldScale + direction * 0.05))

    setZoom(newScale)

    const newPos = {
      x: pointer.x - mousePointTo.x * newScale,
      y: pointer.y - mousePointTo.y * newScale,
    }

    stage.position(newPos)
  }

  return (
    <div className="flex h-screen flex-col">
      <EditorToolbar />

      <div className="flex flex-1 overflow-hidden">
        <EditorSidebar />

        <div className="flex-1 bg-muted/50 p-8">
          <div className="flex h-full items-center justify-center">
            <Stage
              ref={stageRef}
              width={window.innerWidth - 560 - 300} // sidebar + properties
              height={window.innerHeight - 60} // toolbar
              scaleX={zoom}
              scaleY={zoom}
              onClick={handleStageClick}
              onWheel={handleWheel}
            >
              <Layer ref={layerRef}>
                {/* Background */}
                <Rect
                  x={0}
                  y={0}
                  width={canvasWidth}
                  height={canvasHeight}
                  fill={design.canvas.backgroundColor || '#ffffff'}
                  shadowColor="black"
                  shadowBlur={10}
                  shadowOpacity={0.1}
                />

                {/* Render layers */}
                {design.layers
                  .sort((a, b) => (a.order ?? 0) - (b.order ?? 0))
                  .map((layer) => (
                    <KonvaLayerFactory
                      key={layer.id}
                      layer={layer}
                      isSelected={selectedLayerId === layer.id}
                      onSelect={() => selectLayer(layer.id)}
                      onChange={(updates) => updateLayer(layer.id, updates)}
                    />
                  ))}

                {/* Transformer for selected layer */}
                <KonvaTransformer selectedLayerId={selectedLayerId} />
              </Layer>
            </Stage>
          </div>
        </div>

        <PropertiesPanel />
      </div>

      <LayersPanel />
    </div>
  )
}

export function KonvaEditor() {
  return <KonvaEditorCore />
}
```

### KonvaLayerFactory.tsx (Factory Pattern)

```tsx
import * as React from 'react'
import { Text, Image, Rect, Circle, Star } from 'react-konva'
import useImage from 'use-image'
import Konva from 'konva'
import type { Layer } from '@/types/template'

interface KonvaLayerFactoryProps {
  layer: Layer
  isSelected: boolean
  onSelect: () => void
  onChange: (updates: Partial<Layer>) => void
}

export function KonvaLayerFactory({
  layer,
  isSelected,
  onSelect,
  onChange,
}: KonvaLayerFactoryProps) {
  const shapeRef = React.useRef<any>(null)

  const handleDragEnd = (e: any) => {
    onChange({
      position: {
        x: Math.round(e.target.x()),
        y: Math.round(e.target.y()),
      },
    })
  }

  const handleTransformEnd = (e: any) => {
    const node = shapeRef.current
    const scaleX = node.scaleX()
    const scaleY = node.scaleY()

    // Reset scale
    node.scaleX(1)
    node.scaleY(1)

    onChange({
      position: {
        x: Math.round(node.x()),
        y: Math.round(node.y()),
      },
      size: {
        width: Math.max(5, Math.round(node.width() * scaleX)),
        height: Math.max(5, Math.round(node.height() * scaleY)),
      },
      rotation: Math.round(node.rotation()),
    })
  }

  const commonProps = {
    ref: shapeRef,
    x: layer.position.x,
    y: layer.position.y,
    rotation: layer.rotation || 0,
    opacity: layer.visible ? (layer.style?.opacity ?? 1) : 0.3,
    draggable: !layer.locked,
    onClick: onSelect,
    onTap: onSelect,
    onDragEnd: handleDragEnd,
    onTransformEnd: handleTransformEnd,
  }

  switch (layer.type) {
    case 'text':
      return (
        <Text
          {...commonProps}
          text={layer.content || ''}
          fontSize={layer.style?.fontSize || 16}
          fontFamily={layer.style?.fontFamily || 'Inter'}
          fontStyle={layer.style?.fontStyle || 'normal'}
          fill={layer.style?.color || '#000000'}
          align={layer.style?.textAlign || 'left'}
          width={layer.size.width}
        />
      )

    case 'image':
    case 'logo':
    case 'element':
      return <ImageNode layer={layer} commonProps={commonProps} />

    case 'gradient':
    case 'gradient2':
      return <GradientNode layer={layer} commonProps={commonProps} />

    default:
      return null
  }
}

function ImageNode({ layer, commonProps }: any) {
  const [image] = useImage(layer.fileUrl || '', 'anonymous')
  const imageRef = React.useRef<Konva.Image>(null)

  React.useEffect(() => {
    if (image && imageRef.current) {
      imageRef.current.cache()
      imageRef.current.getLayer()?.batchDraw()
    }
  }, [image, layer.style])

  const filters = React.useMemo(() => {
    const filterList = []
    if (layer.style?.blur) filterList.push(Konva.Filters.Blur)
    if (layer.style?.brightness !== undefined) filterList.push(Konva.Filters.Brighten)
    if (layer.style?.contrast !== undefined) filterList.push(Konva.Filters.Contrast)
    if (layer.style?.grayscale) filterList.push(Konva.Filters.Grayscale)
    if (layer.style?.sepia) filterList.push(Konva.Filters.Sepia)
    if (layer.style?.invert) filterList.push(Konva.Filters.Invert)
    return filterList
  }, [layer.style])

  return (
    <Image
      {...commonProps}
      ref={imageRef}
      image={image}
      width={layer.size.width}
      height={layer.size.height}
      filters={filters}
      blurRadius={layer.style?.blur || 0}
      brightness={layer.style?.brightness || 0}
      contrast={layer.style?.contrast || 0}
    />
  )
}

function GradientNode({ layer, commonProps }: any) {
  const stops = layer.style?.gradientStops || []
  const angle = layer.style?.gradientAngle || 0
  const type = layer.style?.gradientType || 'linear'

  const colorStops = React.useMemo(() => {
    return stops.flatMap((stop: any) => [stop.position, stop.color])
  }, [stops])

  if (type === 'radial') {
    return (
      <Rect
        {...commonProps}
        width={layer.size.width}
        height={layer.size.height}
        fillRadialGradientStartPoint={{ x: layer.size.width / 2, y: layer.size.height / 2 }}
        fillRadialGradientStartRadius={0}
        fillRadialGradientEndPoint={{ x: layer.size.width / 2, y: layer.size.height / 2 }}
        fillRadialGradientEndRadius={Math.max(layer.size.width, layer.size.height) / 2}
        fillRadialGradientColorStops={colorStops}
      />
    )
  }

  return (
    <Rect
      {...commonProps}
      width={layer.size.width}
      height={layer.size.height}
      fillLinearGradientStartPoint={{ x: 0, y: 0 }}
      fillLinearGradientEndPoint={{
        x: Math.cos((angle * Math.PI) / 180) * layer.size.width,
        y: Math.sin((angle * Math.PI) / 180) * layer.size.height,
      }}
      fillLinearGradientColorStops={colorStops}
    />
  )
}
```

### KonvaTransformer.tsx (Transform Controls)

```tsx
import * as React from 'react'
import { Transformer } from 'react-konva'
import { useTemplateEditor } from '@/contexts/template-editor-context'

interface KonvaTransformerProps {
  selectedLayerId: string | null
}

export function KonvaTransformer({ selectedLayerId }: KonvaTransformerProps) {
  const transformerRef = React.useRef<any>(null)
  const { design } = useTemplateEditor()

  React.useEffect(() => {
    if (!transformerRef.current) return

    const stage = transformerRef.current.getStage()
    if (!stage) return

    const selectedNode = stage.findOne(`#${selectedLayerId}`)

    if (selectedNode) {
      transformerRef.current.nodes([selectedNode])
      transformerRef.current.getLayer().batchDraw()
    } else {
      transformerRef.current.nodes([])
    }
  }, [selectedLayerId])

  return (
    <Transformer
      ref={transformerRef}
      borderStroke="hsl(var(--primary))"
      borderStrokeWidth={2}
      anchorStroke="hsl(var(--primary))"
      anchorFill="white"
      anchorSize={10}
      anchorCornerRadius={5}
      enabledAnchors={[
        'top-left',
        'top-right',
        'bottom-left',
        'bottom-right',
        'middle-left',
        'middle-right',
        'top-center',
        'bottom-center',
      ]}
      rotateAnchorOffset={30}
      boundBoxFunc={(oldBox, newBox) => {
        // Limit resize
        if (newBox.width < 5 || newBox.height < 5) {
          return oldBox
        }
        return newBox
      }}
    />
  )
}
```

---

## 🎨 Sistema de Templates

### Templates Panel (Sidebar)

```tsx
"use client"

import * as React from 'react'
import Image from 'next/image'
import { useTemplates } from '@/hooks/use-templates'
import { useTemplateEditor } from '@/contexts/template-editor-context'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Tabs, TabsList, TabsTrigger, TabsContent } from '@/components/ui/tabs'
import { Skeleton } from '@/components/ui/skeleton'
import { ScrollArea } from '@/components/ui/scroll-area'

export function TemplatesPanel() {
  const [search, setSearch] = React.useState('')
  const [category, setCategory] = React.useState('all')

  const { data: templates, isLoading } = useTemplates({ category, search })
  const { loadTemplate } = useTemplateEditor()

  const categories = [
    { id: 'all', label: 'Todos' },
    { id: 'social', label: 'Social Media' },
    { id: 'business', label: 'Business Cards' },
    { id: 'flyer', label: 'Flyers' },
    { id: 'poster', label: 'Posters' },
    { id: 'presentation', label: 'Apresentações' },
  ]

  return (
    <div className="flex h-full flex-col gap-4 p-4">
      <div className="space-y-2">
        <h3 className="font-semibold">Templates</h3>
        <Input
          placeholder="Buscar templates..."
          value={search}
          onChange={(e) => setSearch(e.target.value)}
        />
      </div>

      <Tabs value={category} onValueChange={setCategory}>
        <TabsList className="w-full">
          {categories.map((cat) => (
            <TabsTrigger key={cat.id} value={cat.id} className="flex-1">
              {cat.label}
            </TabsTrigger>
          ))}
        </TabsList>
      </Tabs>

      <ScrollArea className="flex-1">
        <div className="grid grid-cols-2 gap-3">
          {isLoading ? (
            Array.from({ length: 6 }).map((_, i) => (
              <Skeleton key={i} className="aspect-[3/4] w-full" />
            ))
          ) : (
            templates?.map((template) => (
              <button
                key={template.id}
                onClick={() => loadTemplate(template.id)}
                className="group relative aspect-[3/4] overflow-hidden rounded-lg border border-border bg-card transition-all hover:border-primary hover:shadow-lg"
              >
                <Image
                  src={template.thumbnailUrl}
                  alt={template.name}
                  fill
                  className="object-cover"
                />
                <div className="absolute inset-0 bg-gradient-to-t from-black/60 to-transparent opacity-0 transition-opacity group-hover:opacity-100">
                  <div className="absolute bottom-2 left-2 right-2 text-left text-sm text-white">
                    {template.name}
                  </div>
                </div>
              </button>
            ))
          )}
        </div>
      </ScrollArea>

      <Button variant="outline" className="w-full">
        Criar Template em Branco
      </Button>
    </div>
  )
}
```

### Template Service

```typescript
// src/lib/assets/templates-service.ts
import { api } from '@/lib/api-client'
import type { DesignData } from '@/types/template'

export interface Template {
  id: number
  name: string
  category: string
  thumbnailUrl: string
  designData: DesignData
  tags: string[]
  isPremium: boolean
  createdAt: string
}

export class TemplatesService {
  static async listTemplates(params?: {
    category?: string
    search?: string
    limit?: number
  }): Promise<Template[]> {
    return api.get('/api/templates', { params })
  }

  static async getTemplate(id: number): Promise<Template> {
    return api.get(`/api/templates/${id}`)
  }

  static async createTemplate(data: {
    name: string
    category: string
    designData: DesignData
    thumbnailUrl?: string
  }): Promise<Template> {
    return api.post('/api/templates', data)
  }

  static async generateThumbnail(designData: DesignData): Promise<string> {
    // Renderiza thumbnail usando RenderEngine
    const canvas = document.createElement('canvas')
    canvas.width = 300
    canvas.height = 400
    const ctx = canvas.getContext('2d')!

    await RenderEngine.renderDesign(ctx, designData, {}, { scaleFactor: 0.3 })

    return canvas.toDataURL('image/png')
  }
}
```

---

## 🖼️ Asset Library System

### Images Panel (Unsplash Integration)

```tsx
"use client"

import * as React from 'react'
import Image from 'next/image'
import { createApi } from 'unsplash-js'
import { Input } from '@/components/ui/input'
import { ScrollArea } from '@/components/ui/scroll-area'
import { Skeleton } from '@/components/ui/skeleton'
import { useTemplateEditor } from '@/contexts/template-editor-context'
import { nanoid } from 'nanoid'

const unsplash = createApi({
  accessKey: process.env.NEXT_PUBLIC_UNSPLASH_ACCESS_KEY!,
})

export function ImagesPanel() {
  const [search, setSearch] = React.useState('business')
  const [images, setImages] = React.useState<any[]>([])
  const [loading, setLoading] = React.useState(false)

  const { addLayer } = useTemplateEditor()

  const searchImages = React.useCallback(async (query: string) => {
    setLoading(true)
    try {
      const result = await unsplash.search.getPhotos({
        query,
        perPage: 20,
        orientation: 'landscape',
      })

      if (result.type === 'success') {
        setImages(result.response.results)
      }
    } catch (error) {
      console.error('Failed to fetch images:', error)
    } finally {
      setLoading(false)
    }
  }, [])

  React.useEffect(() => {
    const timer = setTimeout(() => {
      if (search) searchImages(search)
    }, 500)

    return () => clearTimeout(timer)
  }, [search, searchImages])

  const handleImageClick = (imageUrl: string) => {
    // Add image layer to canvas
    addLayer({
      id: nanoid(),
      type: 'image',
      name: 'Nova Imagem',
      visible: true,
      locked: false,
      order: Date.now(),
      position: { x: 100, y: 100 },
      size: { width: 300, height: 200 },
      rotation: 0,
      fileUrl: imageUrl,
    })
  }

  return (
    <div className="flex h-full flex-col gap-4 p-4">
      <div className="space-y-2">
        <h3 className="font-semibold">Imagens</h3>
        <Input
          placeholder="Buscar no Unsplash..."
          value={search}
          onChange={(e) => setSearch(e.target.value)}
        />
      </div>

      <ScrollArea className="flex-1">
        <div className="grid grid-cols-2 gap-2">
          {loading ? (
            Array.from({ length: 8 }).map((_, i) => (
              <Skeleton key={i} className="aspect-video w-full" />
            ))
          ) : (
            images.map((img) => (
              <button
                key={img.id}
                onClick={() => handleImageClick(img.urls.regular)}
                className="group relative aspect-video overflow-hidden rounded-md border border-border bg-muted transition-all hover:border-primary hover:shadow-md"
              >
                <Image
                  src={img.urls.small}
                  alt={img.alt_description || 'Image'}
                  fill
                  className="object-cover transition-transform group-hover:scale-110"
                />
              </button>
            ))
          )}
        </div>
      </ScrollArea>

      <div className="text-xs text-muted-foreground">
        Powered by{' '}
        <a
          href="https://unsplash.com"
          target="_blank"
          rel="noopener noreferrer"
          className="underline"
        >
          Unsplash
        </a>
      </div>
    </div>
  )
}
```

### Shapes Panel

```tsx
"use client"

import * as React from 'react'
import { useTemplateEditor } from '@/contexts/template-editor-context'
import { ScrollArea } from '@/components/ui/scroll-area'
import { nanoid } from 'nanoid'

const shapes = [
  { id: 'rectangle', label: 'Retângulo', icon: '▭' },
  { id: 'circle', label: 'Círculo', icon: '●' },
  { id: 'triangle', label: 'Triângulo', icon: '▲' },
  { id: 'star', label: 'Estrela', icon: '★' },
  { id: 'polygon', label: 'Polígono', icon: '⬡' },
  { id: 'arrow', label: 'Seta', icon: '➜' },
  { id: 'line', label: 'Linha', icon: '─' },
]

export function ShapesPanel() {
  const { addLayer } = useTemplateEditor()

  const handleShapeClick = (shapeId: string) => {
    addLayer({
      id: nanoid(),
      type: 'shape',
      name: shapes.find(s => s.id === shapeId)?.label || 'Forma',
      visible: true,
      locked: false,
      order: Date.now(),
      position: { x: 200, y: 200 },
      size: { width: 150, height: 150 },
      rotation: 0,
      style: {
        fill: '#3b82f6',
        stroke: '#1e40af',
        strokeWidth: 2,
      },
      shapeType: shapeId,
    })
  }

  return (
    <div className="flex h-full flex-col gap-4 p-4">
      <h3 className="font-semibold">Formas</h3>

      <ScrollArea className="flex-1">
        <div className="grid grid-cols-3 gap-2">
          {shapes.map((shape) => (
            <button
              key={shape.id}
              onClick={() => handleShapeClick(shape.id)}
              className="flex aspect-square flex-col items-center justify-center gap-2 rounded-lg border border-border bg-card text-4xl transition-all hover:border-primary hover:bg-card/80"
            >
              {shape.icon}
              <span className="text-xs">{shape.label}</span>
            </button>
          ))}
        </div>
      </ScrollArea>
    </div>
  )
}
```

---

## 🛠️ Ferramentas de Edição de Imagem

### Image Properties Panel (Com Filtros)

```tsx
"use client"

import * as React from 'react'
import { useTemplateEditor } from '@/contexts/template-editor-context'
import { Label } from '@/components/ui/label'
import { Slider } from '@/components/ui/slider'
import { Button } from '@/components/ui/button'
import { Separator } from '@/components/ui/separator'

export function ImageProperties() {
  const { selectedLayer, updateLayerStyle } = useTemplateEditor()

  if (!selectedLayer || selectedLayer.type !== 'image') return null

  const style = selectedLayer.style || {}

  return (
    <div className="space-y-4">
      <div>
        <h4 className="mb-2 font-semibold">Filtros de Imagem</h4>

        {/* Brightness */}
        <div className="space-y-2">
          <Label>Brilho</Label>
          <Slider
            value={[(style.brightness || 0) * 100 + 50]}
            onValueChange={([value]) =>
              updateLayerStyle(selectedLayer.id, {
                ...style,
                brightness: (value - 50) / 100,
              })
            }
            min={0}
            max={100}
            step={1}
          />
        </div>

        {/* Contrast */}
        <div className="space-y-2">
          <Label>Contraste</Label>
          <Slider
            value={[(style.contrast || 0) * 100 + 50]}
            onValueChange={([value]) =>
              updateLayerStyle(selectedLayer.id, {
                ...style,
                contrast: (value - 50) / 100,
              })
            }
            min={0}
            max={100}
            step={1}
          />
        </div>

        {/* Blur */}
        <div className="space-y-2">
          <Label>Desfoque</Label>
          <Slider
            value={[style.blur || 0]}
            onValueChange={([value]) =>
              updateLayerStyle(selectedLayer.id, {
                ...style,
                blur: value,
              })
            }
            min={0}
            max={20}
            step={0.5}
          />
        </div>

        <Separator className="my-4" />

        {/* Preset Filters */}
        <div className="space-y-2">
          <Label>Filtros Rápidos</Label>
          <div className="grid grid-cols-2 gap-2">
            <Button
              variant="outline"
              size="sm"
              onClick={() =>
                updateLayerStyle(selectedLayer.id, {
                  ...style,
                  grayscale: !style.grayscale,
                })
              }
            >
              Preto e Branco
            </Button>
            <Button
              variant="outline"
              size="sm"
              onClick={() =>
                updateLayerStyle(selectedLayer.id, {
                  ...style,
                  sepia: !style.sepia,
                })
              }
            >
              Sépia
            </Button>
            <Button
              variant="outline"
              size="sm"
              onClick={() =>
                updateLayerStyle(selectedLayer.id, {
                  ...style,
                  invert: !style.invert,
                })
              }
            >
              Inverter
            </Button>
          </div>
        </div>

        <Separator className="my-4" />

        {/* Advanced Editing */}
        <Button variant="default" className="w-full">
          Edição Avançada (Crop, Rotate)
        </Button>
      </div>
    </div>
  )
}
```

### Advanced Image Editor Modal

```tsx
"use client"

import * as React from 'react'
import Cropper from 'cropperjs'
import 'cropperjs/dist/cropper.css'
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogFooter,
} from '@/components/ui/dialog'
import { Button } from '@/components/ui/button'
import { Tabs, TabsList, TabsTrigger, TabsContent } from '@/components/ui/tabs'

interface ImageEditorModalProps {
  open: boolean
  onClose: () => void
  imageUrl: string
  onSave: (editedImageUrl: string) => void
}

export function ImageEditorModal({
  open,
  onClose,
  imageUrl,
  onSave,
}: ImageEditorModalProps) {
  const imageRef = React.useRef<HTMLImageElement>(null)
  const cropperRef = React.useRef<Cropper | null>(null)
  const [tab, setTab] = React.useState('crop')

  React.useEffect(() => {
    if (open && imageRef.current && !cropperRef.current) {
      cropperRef.current = new Cropper(imageRef.current, {
        aspectRatio: NaN,
        viewMode: 1,
        autoCropArea: 1,
      })
    }

    return () => {
      cropperRef.current?.destroy()
      cropperRef.current = null
    }
  }, [open])

  const handleRotate = (degrees: number) => {
    cropperRef.current?.rotate(degrees)
  }

  const handleFlip = (direction: 'horizontal' | 'vertical') => {
    if (direction === 'horizontal') {
      cropperRef.current?.scaleX(-1 * (cropperRef.current.getData().scaleX || 1))
    } else {
      cropperRef.current?.scaleY(-1 * (cropperRef.current.getData().scaleY || 1))
    }
  }

  const handleSave = () => {
    const canvas = cropperRef.current?.getCroppedCanvas()
    if (canvas) {
      const editedImageUrl = canvas.toDataURL('image/png')
      onSave(editedImageUrl)
      onClose()
    }
  }

  return (
    <Dialog open={open} onOpenChange={onClose}>
      <DialogContent className="max-w-4xl">
        <DialogHeader>
          <DialogTitle>Editor de Imagem</DialogTitle>
        </DialogHeader>

        <Tabs value={tab} onValueChange={setTab}>
          <TabsList>
            <TabsTrigger value="crop">Cortar</TabsTrigger>
            <TabsTrigger value="rotate">Girar</TabsTrigger>
            <TabsTrigger value="flip">Espelhar</TabsTrigger>
          </TabsList>

          <TabsContent value="crop" className="space-y-4">
            <div className="relative h-[400px] w-full">
              <img ref={imageRef} src={imageUrl} alt="Edit" />
            </div>
          </TabsContent>

          <TabsContent value="rotate" className="space-y-4">
            <div className="flex gap-2">
              <Button onClick={() => handleRotate(90)}>Girar 90° ↻</Button>
              <Button onClick={() => handleRotate(-90)}>Girar 90° ↺</Button>
              <Button onClick={() => handleRotate(180)}>Girar 180°</Button>
            </div>
            <div className="relative h-[400px] w-full">
              <img ref={imageRef} src={imageUrl} alt="Edit" />
            </div>
          </TabsContent>

          <TabsContent value="flip" className="space-y-4">
            <div className="flex gap-2">
              <Button onClick={() => handleFlip('horizontal')}>
                Espelhar Horizontal ⇄
              </Button>
              <Button onClick={() => handleFlip('vertical')}>
                Espelhar Vertical ⇅
              </Button>
            </div>
            <div className="relative h-[400px] w-full">
              <img ref={imageRef} src={imageUrl} alt="Edit" />
            </div>
          </TabsContent>
        </Tabs>

        <DialogFooter>
          <Button variant="outline" onClick={onClose}>
            Cancelar
          </Button>
          <Button onClick={handleSave}>Salvar Alterações</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  )
}
```

---

## ⏮️ Undo/Redo System

### History Manager

```typescript
// src/lib/history/history-manager.ts
import type { DesignData } from '@/types/template'

interface HistoryState {
  design: DesignData
  timestamp: number
}

export class HistoryManager {
  private past: HistoryState[] = []
  private future: HistoryState[] = []
  private maxHistory = 50

  pushState(design: DesignData): void {
    this.past.push({
      design: JSON.parse(JSON.stringify(design)), // Deep clone
      timestamp: Date.now(),
    })

    // Limit history size
    if (this.past.length > this.maxHistory) {
      this.past.shift()
    }

    // Clear future on new action
    this.future = []
  }

  undo(): DesignData | null {
    if (this.past.length === 0) return null

    const current = this.past.pop()!
    this.future.push(current)

    return this.past[this.past.length - 1]?.design || null
  }

  redo(): DesignData | null {
    if (this.future.length === 0) return null

    const next = this.future.pop()!
    this.past.push(next)

    return next.design
  }

  canUndo(): boolean {
    return this.past.length > 0
  }

  canRedo(): boolean {
    return this.future.length > 0
  }

  clear(): void {
    this.past = []
    this.future = []
  }
}
```

### useHistory Hook

```typescript
// src/hooks/editor/use-history.ts
import * as React from 'react'
import { useTemplateEditor } from '@/contexts/template-editor-context'
import { HistoryManager } from '@/lib/history/history-manager'

export function useHistory() {
  const historyRef = React.useRef(new HistoryManager())
  const { design, setDesign } = useTemplateEditor()

  const saveState = React.useCallback(() => {
    historyRef.current.pushState(design)
  }, [design])

  const undo = React.useCallback(() => {
    const prevState = historyRef.current.undo()
    if (prevState) {
      setDesign(prevState)
    }
  }, [setDesign])

  const redo = React.useCallback(() => {
    const nextState = historyRef.current.redo()
    if (nextState) {
      setDesign(nextState)
    }
  }, [setDesign])

  // Keyboard shortcuts
  React.useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if ((e.ctrlKey || e.metaKey) && e.key === 'z') {
        e.preventDefault()
        if (e.shiftKey) {
          redo()
        } else {
          undo()
        }
      }
    }

    window.addEventListener('keydown', handleKeyDown)
    return () => window.removeEventListener('keydown', handleKeyDown)
  }, [undo, redo])

  return {
    saveState,
    undo,
    redo,
    canUndo: historyRef.current.canUndo(),
    canRedo: historyRef.current.canRedo(),
  }
}
```

---

## 🎯 Fases de Implementação

### **FASE 1: Setup e Konva Core** (3-4 dias)

**Objetivos:**
- Configurar Konva.js e react-konva
- Implementar componente base funcional
- Resolver SSR issues com Next.js 15

**Tarefas:**
1. ✅ Instalar dependências
   ```bash
   npm install konva react-konva react-konva-utils use-image
   npm install -D @types/konva
   ```

2. ✅ Configurar `next.config.ts` com webpack externals

3. ✅ Criar `KonvaEditor` com dynamic import
   - Wrapper com `ssr: false`
   - Stage básico funcionando

4. ✅ Implementar `KonvaLayerFactory` para text + image layers

5. ✅ Testar renderização básica

**Critérios de Sucesso:**
- [ ] Konva Stage renderiza sem erros de SSR
- [ ] Text e Image layers aparecem no canvas
- [ ] Build production passa sem warnings

---

### **FASE 2: Transformer e Interatividade** (3-4 dias)

**Objetivos:**
- Adicionar controles de transformação (resize, rotate)
- Implementar drag & drop
- Sincronização com contexto

**Tarefas:**
1. ✅ Implementar `KonvaTransformer`
   - Resize handles
   - Rotation handle
   - Bound box constraints

2. ✅ Configurar event handlers
   - `onDragEnd` → updatePosition
   - `onTransformEnd` → updateSize/rotation
   - `onClick` → selectLayer

3. ✅ Sincronização bidirecional
   - Konva → Design Data
   - Properties Panel → Konva

4. ✅ Testar performance com 20+ layers

**Critérios de Sucesso:**
- [ ] Resize funciona com handles visuais
- [ ] Rotation smooth e preciso
- [ ] Drag & drop fluido (60 FPS)
- [ ] Sync perfeito com properties panel

---

### **FASE 3: Sistema de Templates** (4-5 dias)

**Objetivos:**
- Criar biblioteca de templates
- Implementar categorização e busca
- One-click template loading

**Tarefas:**
1. ✅ Criar API `/api/templates`
   - List templates (com filtros)
   - Get template by ID
   - Create template

2. ✅ Implementar `TemplatesPanel`
   - Grid de thumbnails
   - Categorias (tabs)
   - Search input

3. ✅ Gerar thumbnails automaticamente
   - Usar RenderEngine para preview
   - Salvar no banco/storage

4. ✅ Integrar com editor
   - `loadTemplate()` method no contexto
   - Substituir design atual

**Critérios de Sucesso:**
- [ ] Templates carregam em <500ms
- [ ] Thumbnails renderizam corretamente
- [ ] Search funciona em tempo real
- [ ] Categorias filtram corretamente

---

### **FASE 4: Asset Library** (5-6 dias)

**Objetivos:**
- Integrar Unsplash para imagens
- Criar biblioteca de shapes e ícones
- Upload de assets customizados

**Tarefas:**
1. ✅ Implementar `ImagesPanel` (Unsplash)
   - Search API
   - Grid de resultados
   - Drag to canvas

2. ✅ Implementar `ShapesPanel`
   - Formas pré-definidas (rect, circle, star, etc.)
   - Click to add to canvas

3. ✅ Implementar `IconsPanel`
   - Font Awesome integration
   - SVG icons library

4. ✅ Implementar `UploadsPanel`
   - File upload (drag & drop)
   - Image compression
   - Storage (Google Drive ou local)

5. ✅ Implementar `BackgroundsPanel`
   - Gradients presets
   - Patterns
   - Solid colors

**Critérios de Sucesso:**
- [ ] Unsplash search retorna resultados em <1s
- [ ] Shapes adicionam ao canvas em posição centralizada
- [ ] Upload suporta PNG, JPG, SVG
- [ ] Background aplicam ao canvas corretamente

---

### **FASE 5: Ferramentas de Edição de Imagem** (5-6 dias)

**Objetivos:**
- Implementar filtros de imagem
- Criar editor avançado (crop, rotate, flip)
- Adicionar presets de filtros

**Tarefas:**
1. ✅ Implementar filtros Konva
   - Blur, brightness, contrast
   - Grayscale, sepia, invert
   - Integrar com Properties Panel

2. ✅ Criar `ImageEditorModal`
   - Crop com Cropper.js
   - Rotate (90°, -90°, 180°)
   - Flip (horizontal, vertical)

3. ✅ Adicionar presets de filtros
   - "Vintage", "Cold", "Warm", "Dramatic"
   - One-click apply

4. ✅ Implementar background removal (opcional)
   - Integração com remove.bg API
   - Fallback manual com alpha mask

**Critérios de Sucesso:**
- [ ] Filtros aplicam em tempo real
- [ ] Crop funciona com preview
- [ ] Rotate/flip preservam qualidade
- [ ] Export mantém edições aplicadas

---

### **FASE 6: Interface Canva-like** (4-5 dias)

**Objetivos:**
- Refinar UI para parecer com Canva
- Adicionar toolbar superior
- Implementar layers panel inferior
- Polir micro-interações

**Tarefas:**
1. ✅ Redesign `EditorToolbar`
   - Save, Export, Share buttons
   - Undo/Redo buttons
   - Zoom controls

2. ✅ Criar `EditorSidebar` com tabs
   - Templates, Text, Images, Shapes, Icons, Uploads, BG
   - Collapsible/expandable

3. ✅ Refinar `PropertiesPanel`
   - Context-sensitive (muda conforme layer selecionado)
   - Grouped properties (Position, Style, Effects)

4. ✅ Implementar `LayersPanel` (inferior, collapsible)
   - Drag to reorder
   - Eye icon (visibility)
   - Lock icon

5. ✅ Adicionar micro-animações
   - Framer Motion para transições
   - Hover effects
   - Loading states

**Critérios de Sucesso:**
- [ ] UI parece profissional e polida
- [ ] Navigation intuitiva
- [ ] Responsivo (funciona em tablets)
- [ ] Feedback visual em todas ações

---

### **FASE 7: Zoom, Pan e Undo/Redo** (3-4 dias)

**Objetivos:**
- Implementar zoom/pan profissional
- Sistema de histórico completo
- Keyboard shortcuts

**Tarefas:**
1. ✅ Implementar zoom com mouse wheel
   - Zoom to pointer position
   - Min 25%, Max 200%

2. ✅ Implementar pan (spacebar + drag)
   - Cursor muda para "grab"
   - Smooth dragging

3. ✅ Criar `HistoryManager`
   - Undo/redo stack (max 50 states)
   - Deep clone de design data

4. ✅ Adicionar keyboard shortcuts
   - Ctrl+Z/Y: Undo/Redo
   - Ctrl+C/V: Copy/Paste
   - Delete: Remove layer
   - Arrows: Nudge position

**Critérios de Sucesso:**
- [ ] Zoom suave e preciso
- [ ] Pan não interfere com seleção
- [ ] Undo/redo funciona perfeitamente
- [ ] Shortcuts intuitivos

---

### **FASE 8: Export e Performance** (3-4 dias)

**Objetivos:**
- Implementar export profissional (PNG, JPG)
- Implementar nova aba (lado direito junto com as propriedades ) criativos com as exports para visualização com lightbox
- Otimizar performance para templates grandes
- Adicionar loading states

**Tarefas:**
1. ✅ Implementar export via Konva
   ```typescript
   const dataURL = stageRef.current.toDataURL({
     pixelRatio: 2, // 2x resolution
     mimeType: 'image/png',
   })
   ```

2. ✅ Otimizar performance
   - Object caching (`layer.cache()`)
   - Lazy rendering
   - Virtualized layers panel

3. ✅ Adicionar loading states
   - Skeleton loaders
   - Progress indicators
   - Optimistic UI updates

**Critérios de Sucesso:**
- [ ] Export PNG em alta resolução
- [ ] Performance suave com 50+ layers
- [ ] Loading states em todas operações assíncronas

---

### **FASE 9: Preview em Tempo Real** (2-3 dias)

**Objetivos:**
- Canvas já É o preview (WYSIWYG)
- Adicionar preview modal para export
- Preview responsivo (diferentes tamanhos)

**Tarefas:**
1. ✅ Canvas como WYSIWYG
   - O que você vê é o que você exporta
   - Renderização idêntica ao backend

2. ✅ Criar `PreviewModal`
   - Preview antes de export
   - Diferentes formatos (PNG, JPG, PDF)
   - Ajuste de qualidade

3. ✅ Preview responsivo
   - Mobile preview
   - Tablet preview
   - Desktop preview

**Critérios de Sucesso:**
- [ ] Canvas representa 100% do export final
- [ ] Preview modal mostra resultado exato
- [ ] Responsive preview funciona

---

### **FASE 10: Polish e Testes** (3-4 dias)

**Objetivos:**
- Refinar UX baseado em testes
- Corrigir bugs edge cases
- Documentar código

**Tarefas:**
1. ✅ Testes de usabilidade
   - Usuários não-técnicos testam editor
   - Coletar feedback

2. ✅ Corrigir bugs encontrados
   - Edge cases
   - Performance issues

3. ✅ Documentação
   - JSDoc em componentes principais
   - README do editor
   - Guia de uso

4. ✅ Accessibility
   - Keyboard navigation
   - ARIA labels
   - Screen reader support (básico)

**Critérios de Sucesso:**
- [ ] Zero bugs críticos
- [ ] Performance consistente
- [ ] Código documentado
- [ ] Acessibilidade básica implementada

---

## 📊 Estimativa de Tempo Total

| Fase | Duração | Dias Úteis | Responsável |
|------|---------|------------|-------------|
| **Fase 1** - Setup Konva Core | 3-4 dias | D1-D4 | Dev Backend |
| **Fase 2** - Transformer | 3-4 dias | D5-D8 | Dev Frontend |
| **Fase 3** - Templates System | 4-5 dias | D9-D13 | Dev Fullstack |
| **Fase 4** - Asset Library | 5-6 dias | D14-D19 | Dev Frontend |
| **Fase 5** - Image Editing | 5-6 dias | D20-D25 | Dev Frontend |
| **Fase 6** - Canva-like UI | 4-5 dias | D26-D30 | Designer + Dev |
| **Fase 7** - Zoom/Pan/History | 3-4 dias | D31-D34 | Dev Frontend |
| **Fase 8** - Export/Performance | 3-4 dias | D35-D38 | Dev Backend |
| **Fase 9** - Preview Real-Time | 2-3 dias | D39-D41 | Dev Fullstack |
| **Fase 10** - Polish/Testes | 3-4 dias | D42-D45 | Team |
| **Buffer** | 5 dias | D46-D50 | - |

**TOTAL: ~8-10 semanas** (2 desenvolvedores full-time)

---

## ⚠️ Riscos e Mitigações

### Risco 1: Next.js 15 SSR Issues com Konva
**Probabilidade:** Média
**Impacto:** Médio
**Mitigação:**
- Usar dynamic import com `ssr: false` (solução comprovada)
- Testar em ambiente staging antes de prod
- Ter fallback para versão anterior do editor

### Risco 2: Performance com 100+ layers
**Probabilidade:** Alta
**Impacto:** Alto
**Mitigação:**
- Implementar object caching agressivo
- Virtualized rendering para layers panel
- Limitar número de layers por template (100 max)
- Adicionar warning quando approaching limit

### Risco 3: Unsplash API rate limits
**Probabilidade:** Média
**Impacto:** Médio
**Mitigação:**
- Cache de resultados de busca
- Implementar debouncing (500ms)
- Fallback para Google Drive images
- Considerar Unsplash+ (paid tier) se necessário

### Risco 4: Image filters degradam performance
**Probabilidade:** Média
**Impacto:** Médio
**Mitigação:**
- Cache de filtered images
- Aplicar filtros apenas em selected layer
- Usar Web Workers para heavy filtering (futuro)

### Risco 5: Browser compatibility (Safari)
**Probabilidade:** Baixa
**Impacto:** Médio
**Mitigação:**
- Testar em Safari desde Fase 1
- Polyfills para features faltantes
- Graceful degradation

### Risco 6: Complexidade de UX confunde usuários
**Probabilidade:** Média
**Impacto:** Alto
**Mitigação:**
- Testes de usabilidade frequentes
- Tooltips e onboarding tutorial
- Keyboard shortcuts help modal
- Documentação/tutoriais em vídeo

---

## ✅ Critérios de Aceitação Final

### Funcionalidades Core
- [ ] **Templates**: Biblioteca com 20+ templates prontos
- [ ] **Assets**: Unsplash integration + 50+ shapes/icons
- [ ] **Drag & Drop**: Natural e fluido (60 FPS)
- [ ] **Resize/Rotate**: Handles visuais profissionais
- [ ] **Image Editing**: Filters, crop, rotate, flip funcionam
- [ ] **Undo/Redo**: Stack de 50 estados, Ctrl+Z/Y
- [ ] **Zoom/Pan**: Mouse wheel zoom + spacebar pan
- [ ] **Export**: PNG (2x), JPG, PDF com qualidade
- [ ] **Preview**: WYSIWYG total, canvas = export

### Interface
- [ ] **Canva-like**: UI polida e intuitiva
- [ ] **Sidebar**: Templates, Text, Images, Shapes, Icons, Uploads, BG
- [ ] **Toolbar**: Save, Export, Undo/Redo, Zoom controls
- [ ] **Properties Panel**: Context-sensitive, grouped props
- [ ] **Layers Panel**: Drag to reorder, visibility, lock
- [ ] **Responsivo**: Funciona em desktop + tablet (1024px+)

### Performance
- [ ] **Loading**: Templates carregam < 500ms
- [ ] **Rendering**: 60 FPS com 50 layers
- [ ] **Export**: PNG 2x em < 3s
- [ ] **Memory**: Zero leaks após 30min uso

### Qualidade de Código
- [ ] **TypeScript**: 100% type-safe, zero `any`
- [ ] **Documentação**: JSDoc em components principais
- [ ] **Testes**: Coverage > 70% (opcional mas desejável)
- [ ] **Accessibility**: ARIA labels, keyboard nav

---

## 🚀 Próximos Passos Pós-Lançamento

### V2.1 - Collaboration (Futuro)
- Real-time collaborative editing
- Cursor tracking
- Comments system
- Version control

### V2.2 - AI Features
- AI-powered template suggestions
- Smart object removal (background removal)
- Auto-layout suggestions
- Text-to-image generation

### V2.3 - Advanced Tools
- Vector editing (SVG)
- Animation timeline
- Video editing
- 3D objects support

### V2.4 - Marketplace
- User-generated templates
- Asset marketplace
- Premium templates
- Revenue sharing

---

## 📚 Recursos e Referências

### Documentação Oficial
- [Konva.js Docs](https://konvajs.org/docs/)
- [React-Konva GitHub](https://github.com/konvajs/react-konva)
- [Konva.js Demos](https://konvajs.org/docs/sandbox/index.html)

### Inspiração de Produto
- [Canva.com](https://www.canva.com) - Referência de UX
- [Polotno SDK](https://polotno.com/) - Editor baseado em Konva
- [Figma](https://www.figma.com) - Performance e colaboração

### Bibliotecas Auxiliares
- [Cropper.js](https://fengyuanchen.github.io/cropperjs/) - Image cropping
- [Unsplash API](https://unsplash.com/developers) - Free images
- [dnd-kit](https://dndkit.com/) - Drag and drop
- [jsPDF](https://github.com/parallax/jsPDF) - PDF generation

### Comunidade
- [Konva.js Discord](https://discord.gg/8FqZwVT)
- [Stack Overflow - konva tag](https://stackoverflow.com/questions/tagged/konva)
- [React-Konva Examples](https://codesandbox.io/examples/package/react-konva)

---

## 📝 Checklist de Início

Antes de iniciar Fase 1:

- [ ] Next.js 15.3.5 e React 19 confirmados
- [ ] TypeScript configurado
- [ ] Editor atual funcional (baseline)
- [ ] Branch `feature/konva-editor` criada
- [ ] Unsplash API key obtida (`NEXT_PUBLIC_UNSPLASH_ACCESS_KEY`)
- [ ] Design mockups aprovados
- [ ] Time alinhado (8-10 semanas)
- [ ] Staging environment disponível

---

## 🎯 Conclusão

Este plano transforma o editor do Studio Lagosta em uma plataforma profissional de design visual, competindo diretamente com editores comerciais como Canva. Usando **Konva.js**, conseguimos:

✅ **Performance nativa** de canvas
✅ **React-first architecture** com react-konva
✅ **Sistema completo de templates** categorizado
✅ **Asset library rica** (Unsplash + shapes + icons)
✅ **Ferramentas de edição profissionais** (filters, crop, rotate)
✅ **Interface Canva-like** polida e intuitiva
✅ **Preview em tempo real** (WYSIWYG)
✅ **Zero custo de licenciamento** (MIT License)

**Diferencial Competitivo:**
- Integração nativa com nosso backend (RenderEngine)
- Dynamic fields para automação
- Google Drive integration
- Customizável 100% (white-label potential)

**ROI Esperado:**
- Aumento de 3x na adoção de usuários
- Redução de 80% no tempo de criação de designs
- Capacidade de cobrar premium pela ferramenta
- Marketplace de templates (receita recorrente)

---

**Próximo Passo Imediato:**
Executar **FASE 1** (Setup e Konva Core) e validar renderização básica funcional.

---

**Criado por:** Claude Code
**Data:** Outubro 2025
**Versão:** 2.0 (Konva.js Edition)
**Status:** 🟢 Pronto para Implementação
