# Prompt: ElevenLabs Text‑to‑Dialogue (Podcast / Diálogos)

Start Here
- Leia AGENTS.md e agents/README.md para padrões e checklists.
- Consulte docs/README.md (frontend, backend, uploads, créditos).
- Use os guias: agents/architecture-planning.md, agents/frontend-development.md, agents/backend-development.md, agents/database-development.md, agents/security-check.md.

Objetivo
- Criar uma nova UI (separada do chat) para compor roteiros de podcasts/diálogos em texto e converter para áudio usando a API Text‑to‑Dialogue da ElevenLabs.
- Permitir múltiplos speakers, vozes e configurações por trecho, produzindo um único áudio final.

Referências
- ElevenLabs Text‑to‑Dialogue: https://elevenlabs.io/docs/api-reference/text-to-dialogue/convert

Configuração
- Variáveis (.env.local): ELEVENLABS_API_KEY (e defaults opcionais de voice/model por speaker).
- Dependências: fetch nativo; para armazenar resultado use @vercel/blob.

Arquitetura
- Nova UI: src/app/(protected)/podcast-studio/page.tsx
  - Builder visual: lista de falas (speaker, voiceId, texto, pausa, estilo), ordenáveis; preview de cada fala.
  - Ações: salvar rascunho, gerar áudio final, baixar/compartilhar.
- Nova API: src/app/api/ai/dialogue/route.ts (POST)
  - Body (Zod): { title?: string; segments: Array<{ speaker: string; text: string; voiceId?: string; style?: string; pauseMs?: number }>; format?: 'mp3'|'wav'; }
  - Auth: Clerk; valida tenancy; rate‑limit; debita créditos.
  - Chamada REST: POST na API de dialogue da ElevenLabs; resposta binária de áudio.
  - Resposta: áudio direto (Content-Type) ou URL no Blob.

Modelagem (opcional, se persistir projetos)
- DialogueProject: { id, userId/workspaceId, title, settings(Json), createdAt, updatedAt }
- DialogueSegment: { id, projectId, ordinal, speaker, text, voiceId?, style?, pauseMs? }

Créditos
- Defina FeatureKey dedicado, ex.: 'ai_text_to_dialogue' (custo maior que TTS).
- Atualize src/lib/credits/feature-config.ts e enum OperationType (migração) e use validateCreditsForFeature/deductCreditsForFeature.

Esqueleto de rota (exemplo)
```ts
// src/app/api/ai/dialogue/route.ts
import { NextResponse } from 'next/server'
import { z } from 'zod'
import { auth } from '@clerk/nextjs'
import { validateCreditsForFeature, deductCreditsForFeature } from '@/lib/credits/deduct'
import { type FeatureKey } from '@/lib/credits/feature-config'

const schema = z.object({
  title: z.string().optional(),
  format: z.enum(['mp3','wav']).default('mp3').optional(),
  segments: z.array(z.object({
    speaker: z.string().min(1),
    text: z.string().min(1).max(5000),
    voiceId: z.string().optional(),
    style: z.string().optional(),
    pauseMs: z.number().min(0).max(10000).optional(),
  })).min(1)
})

export async function POST(req: Request) {
  const { userId } = auth()
  if (!userId) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  const input = schema.parse(await req.json())

  const feature: FeatureKey = 'ai_text_to_dialogue' as any // defina no feature-config.ts
  await validateCreditsForFeature(userId, feature)

  const resp = await fetch('https://api.elevenlabs.io/v1/text-to-dialogue/convert', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.ELEVENLABS_API_KEY}`,
      'Content-Type': 'application/json',
      'Accept': `audio/${input.format ?? 'mp3'}`,
    },
    body: JSON.stringify({
      title: input.title,
      dialogue: input.segments.map((s) => ({
        speaker: s.speaker,
        text: s.text,
        voice_id: s.voiceId,
        style: s.style,
        // dependendo da API, pausas podem ser marcadas como texto especial ou metadado
      })),
    }),
  })

  if (!resp.ok) {
    return NextResponse.json({ error: 'ElevenLabs error' }, { status: 502 })
  }

  await deductCreditsForFeature({ clerkUserId: userId, feature, details: { segments: input.segments.length } })
  const arrayBuf = await resp.arrayBuffer()
  return new NextResponse(arrayBuf, { headers: { 'Content-Type': `audio/${input.format ?? 'mp3'}` } })
}
```

UI (MVP)
- Página com editor de falas: grid/lista de segmentos com drag‑and‑drop, seleção de voiceId por speaker, estilo, pauseMs.
- Preview por segmento (usar endpoint TTS opcionalmente para preview rápido por fala).
- Botão "Gerar diálogo" chama a API e exibe player + download; opção de salvar no Blob.

Uploads/Armazenamento
- Recomendado salvar o resultado no Vercel Blob: POST /api/upload (já existente) ou rota dedicada que receba o ArrayBuffer e faça upload server‑side.

Segurança
- Não expor ELEVENLABS_API_KEY; tudo via server route.
- Limitar tamanho total do script e número de segmentos; adicionar rate‑limit por usuário.
- Multi‑tenant: escopar projetos por userId/workspaceId; prevenir vazamentos.

Testes (Manual)
- Criar 2 projetos com vozes/speakers distintos; gerar e ouvir.
- Simular erro (API key incorreta) e validar 502 sem débito.
- Validar débito ao sucesso e latência aceitável.

Entregáveis do PR
- UI /podcast-studio com editor de segmentos.
- API route de dialogue com validação Zod e respostas de áudio.
- Integração de créditos com novo FeatureKey.
- Documentação curta (docs/dialogue.md) e atualização de .env.example.

