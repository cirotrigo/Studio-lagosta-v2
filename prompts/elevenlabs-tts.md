# Prompt: Integração ElevenLabs Text‑to‑Speech (TTS)

Start Here
- Leia AGENTS.md e agents/README.md para padrões e checklists.
- Consulte docs/README.md (backend, frontend, auth, database, uploads).
- Siga os guias: agents/architecture-planning.md, agents/backend-development.md, agents/frontend-development.md, agents/database-development.md, agents/security-check.md.

Objetivo
- Integrar a API de Text‑to‑Speech da ElevenLabs para converter texto em áudio e disponibilizar via UI e/ou link (Vercel Blob).
- Cobrar créditos por geração de áudio (FeatureKey dedicado) e manter logs mínimos de uso.

Referências
- ElevenLabs TTS: https://elevenlabs.io/docs/api-reference/text-to-speech/convert

Configuração
- Variáveis (.env.local):
  - ELEVENLABS_API_KEY
  - (opcional) ELEVENLABS_MODEL_ID (ex.: "eleven_multilingual_v2")
  - (opcional) ELEVENLABS_DEFAULT_VOICE_ID
- Dependências: nenhuma obrigatória além de fetch nativo; para uploads use @vercel/blob.

Arquitetura
- Nova API: src/app/api/ai/tts/route.ts (POST)
  - Body (Zod): { text: string; voiceId?: string; modelId?: string; format?: 'mp3'|'wav'|'ogg'; speed?: number; }
  - Auth: Clerk (usuário logado). Debitar créditos antes de chamar a ElevenLabs.
  - Chamada REST: POST para endpoint de conversão da ElevenLabs com headers Authorization: Bearer ELEVENLABS_API_KEY; retorna áudio binário.
  - Resposta: retornar áudio diretamente (Content-Type) ou salvar no Blob e responder com URL.
- UI (MVP): src/app/(protected)/tools/tts/page.tsx
  - Form com textarea (text), selects (voice/model/format), slider (speed), botão "Gerar".
  - Exibir player de áudio e opção de download; registrar toasts/erros.

Créditos
- Defina um FeatureKey novo, ex.: 'ai_text_to_speech'.
- Atualize src/lib/credits/feature-config.ts (FEATURE_CREDIT_COSTS + mapping OperationType) e o enum OperationType no Prisma (migração).
- Use validateCreditsForFeature e deductCreditsForFeature na API de TTS.

Esqueleto de rota (exemplo)
```ts
// src/app/api/ai/tts/route.ts
import { NextResponse } from 'next/server'
import { z } from 'zod'
import { auth } from '@clerk/nextjs'
import { validateCreditsForFeature, deductCreditsForFeature } from '@/lib/credits/deduct'
import { type FeatureKey } from '@/lib/credits/feature-config'

const schema = z.object({
  text: z.string().min(1).max(5000),
  voiceId: z.string().optional(),
  modelId: z.string().optional(),
  format: z.enum(['mp3','wav','ogg']).default('mp3').optional(),
  speed: z.number().min(0.5).max(2).optional(),
})

export async function POST(req: Request) {
  const { userId } = auth()
  if (!userId) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  const body = await req.json()
  const { text, voiceId, modelId, format = 'mp3', speed } = schema.parse(body)

  const feature: FeatureKey = 'ai_text_to_speech' as any // defina no feature-config.ts
  await validateCreditsForFeature(userId, feature)

  const resp = await fetch(`https://api.elevenlabs.io/v1/text-to-speech/${voiceId ?? process.env.ELEVENLABS_DEFAULT_VOICE_ID ?? 'placeholder'}/convert`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.ELEVENLABS_API_KEY}`,
      'Content-Type': 'application/json',
      'Accept': `audio/${format}`,
    },
    body: JSON.stringify({
      text,
      model_id: modelId ?? process.env.ELEVENLABS_MODEL_ID,
      voice_settings: speed ? { speed } : undefined,
    }),
  })

  if (!resp.ok) {
    return NextResponse.json({ error: 'ElevenLabs error' }, { status: 502 })
  }

  await deductCreditsForFeature({ clerkUserId: userId, feature, details: { format, len: text.length } })
  const arrayBuf = await resp.arrayBuffer()
  return new NextResponse(arrayBuf, { headers: { 'Content-Type': `audio/${format}` } })
}
```

Admin (opcional)
- Página para gerenciar voices e defaults (model/voice/speed). Persistir em AdminSettings (JSON) ou tabela dedicada.

Segurança
- Não expor ELEVENLABS_API_KEY no cliente. Todas as chamadas devem passar pela API route.
- Limitar tamanho do texto e rate‑limit por usuário.

Testes (Manual)
- Gerar áudio curto e validar reprodução no player.
- Forçar erro (API key inválida) e validar resposta 502 sem débito de créditos.
- Validar débito de créditos quando sucesso.

Entregáveis do PR
- API route funcional com validação Zod.
- UI em /tools/tts com player.
- Atualização do sistema de créditos para o novo FeatureKey.
- Documentação curta (docs/tts.md) e .env.example atualizado.

