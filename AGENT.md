# 🏞️ AGENT.md — Arquitetura de Sistemas: BNP Wildlife App

> **Disciplina:** Arquitetura de Sistemas — Prof. Kiev Gama  
> **Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra  
> **Referência metodológica:** [Design-First Collaboration — Rahul Garg / Fowler](https://martinfowler.com/articles/reduce-friction-ai/design-first-collaboration.html)  
> **Status atual:** ✅ Níveis 1–5 concluídos — aguardando opiniões do grupo

---

## 📁 Estrutura do Projeto

```
arquitetura_software/
│
├── AGENT.md                          ← este arquivo (guia e índice geral)
│
├── 00_referencias/                   ← material de entrada (não editável)
│   ├── desc_atividade.txt            → enunciado da atividade (Prof. Kiev)
│   ├── blog.txt                      → artigo Design-First (Rahul Garg / Fowler)
│   ├── Designing with AI_...pdf      → especificação do app BNP
│   └── Proposta de Arquitetura...pdf → proposta da semana anterior (referência)
│
├── 01_design/                        ← decisões progressivas aprovadas pelo grupo
│   ├── nivel_1_capacidades.md        ✅ Capacidades, escopo e SLOs
│   ├── nivel_2_componentes.md        ✅ Lista definitiva de componentes + C4 textual
│   ├── nivel_3_interacoes.md         ✅ Padrões de comunicação + 4 diagramas Mermaid
│   └── nivel_4_contratos.md          ✅ 9 endpoints de API + 5 schemas de eventos
│
├── 02_implementacao/                 ← artefatos técnicos finais
│   ├── stack_tecnologica.md          ✅ Stack completa com configurações
│   ├── diagramas/
│   │   └── diagrama_c4_completo.md  ✅ C4 Context + Container + Component + Deploy
│   └── adrs/
│       ├── adr_001_event_broker.md   ✅ Kafka vs RabbitMQ
│       ├── adr_002_database_strategy.md ✅ PostgreSQL+PostGIS + MongoDB
│       ├── adr_003_realtime_protocol.md ✅ SSE vs WebSocket
│       ├── adr_004_mobile_platform.md   ✅ Nativo vs Cross-platform
│       └── adr_005_backend_language.md  ✅ Go vs Node.js vs Python
│
└── 03_entregaveis/
    └── comparativo_propostas.md      ⚠️ Esqueleto pronto — opiniões do grupo pendentes
```

---

## 🎯 Missão Concluída — Resumo das Decisões

Todo o processo seguiu a metodologia Design-First em 5 níveis. **Nenhuma tecnologia foi escolhida antes que capacidades, componentes, interações e contratos fossem aprovados pelo grupo.**

---

### Nível 1 — Capacidades ✅
| Decisão | Escolha |
|---------|---------|
| Escopo geográfico | App funciona **apenas dentro do parque** (geo-fenced) |
| SLO de avistamentos | **≤ 30 segundos** do registro à plotagem no mapa |
| Botão de pânico offline | **GPS local + fila offline** com reenvio automático ao reconectar |
| Arquitetura de clientes | **Um único app** com UI baseada em role |
| Autenticação | **Mesmo app**, perfis diferentes via login |
| Sistemas legados | **Greenfield** — nenhuma integração na v1 |

**Capacidades IN SCOPE:** Cadastro, informações geográficas, registro de avistamentos, mapa ao vivo, notificações por espécie, controle de aglomeração, botão de pânico, despacho de emergência.

---

### Nível 2 — Componentes ✅

10 componentes definidos com responsabilidade única:

| Componente | Responsabilidade |
|------------|-----------------|
| Mobile App | UI role-based (VISITOR / GUIDE / MEDICAL_STAFF / CONTROL_OFFICER) |
| API Gateway | Entrada única: auth, roteamento, rate limiting, SSE /co-feed |
| Auth & User Service | Cadastro, login, tokens JWT com role claims |
| Park Information Service | Dados de ecossistema por coordenadas GPS |
| Location Tracker Service | Recebe lotes de GPS, mantém posição atual de todos no parque |
| Sighting Service | Recebe e persiste avistamentos como PENDING |
| Crowding Prevention Engine | Conta veículos no raio, decide APPROVED ou SUPPRESSED |
| Map Service | Serve tiles cartográficas; suporte a download offline |
| Notification Service | Assinaturas por espécie; filas CRITICAL e STANDARD |
| Emergency & Control Service | Ciclo completo de emergência — **isolado com recursos dedicados** |

---

### Nível 3 — Interações ✅

| Decisão | Escolha |
|---------|---------|
| Padrão geral | **Híbrido**: síncrono para leituras/crítico, assíncrono para pipeline de eventos |
| Envio de localização | **Lote a cada 30s** (array de posições do período) |
| Atualização do mapa | **Polling a cada ~15s** (dentro do SLO de 30s) |
| Botão de pânico | **Barramento de eventos fila CRITICAL** com consumidor prioritário |
| Feed do CONTROL_OFFICER | **Canal dedicado** (SSE persistente com feed completo do parque) |

**4 fluxos documentados com diagramas Mermaid:**
1. Informações Gerais (GPS → Park Info)
2. Avistamento + Crowding + Notificação
3. Botão de Pânico → Despacho → Coordenação
4. Envio de Localização em Lote (background)

---

### Nível 4 — Contratos ✅

| Decisão | Escolha |
|---------|---------|
| Token | Auto-suficiente: `userId`, `role`, `vehiclePlate`, `expiresAt` embutidos |
| Tipo de animal | `speciesId` referenciando catálogo carregado do backend |
| Localização em lote | Array de todas as posições do período (trajetória dos 30s) |
| Resposta de emergência | Síncrono até despacho: resposta inclui equipes + ETA |
| Polling do mapa | Retorna todos os avistamentos desde timestamp; app filtra localmente |

**9 endpoints definidos:** `/auth/register`, `/auth/login`, `/species`, `/park-info`, `/sightings` (POST+GET), `/subscriptions`, `/locations/batch`, `/emergency` (POST+PUT), `/map/tiles`, `/map/manifest`

**5 eventos do barramento:** `sighting.submitted`, `sighting.approved`, `sighting.suppressed`, `emergency.raised [CRITICAL]`, `emergency.dispatched [CRITICAL]`

---

### Nível 5 — Implementação ✅

| Camada | Tecnologia | Justificativa principal |
|--------|-----------|------------------------|
| **Barramento** | Apache Kafka | Retenção de eventos + tópicos CRITICAL/STANDARD separados |
| **BD relacional/geo** | PostgreSQL 16 + PostGIS | Queries `ST_DWithin` para crowding; `<->` para nearest-staff |
| **BD documentos** | MongoDB 7 | Schema flexível para avistamentos e logs de emergência |
| **Real-time CO** | Server-Sent Events (SSE) | Feed unidirecional; reconexão automática nativa |
| **Mobile** | Swift (iOS) + Kotlin (Android) nativos | GPS background confiável + botão de pânico sem dependência de terceiros |
| **Backend** | Go (Golang) | Goroutines para Location Tracker; pausas GC mínimas no Emergency Service |
| **Orquestração** | Kubernetes | Emergency Service em namespace isolado com `PriorityClass: high-priority` |

---

## ⚠️ Única Pendência: Opiniões do Grupo

O arquivo [`03_entregaveis/comparativo_propostas.md`](03_entregaveis/comparativo_propostas.md) já contém:
- Tabela comparativa entre proposta anterior e nova proposta
- Análise detalhada de 5 diferenças arquiteturais críticas
- Avaliação de qualidade em 7 critérios
- **Seção em branco para cada integrante escrever sua opinião** (requisito do professor — sem IA)

**6 perguntas norteadoras no documento:**
1. A abordagem Design-First mudou a qualidade das decisões?
2. Quais decisões foram as mais difíceis?
3. Qual o risco de gerar arquitetura com um único prompt?
4. O Location Tracker "invisível" é um exemplo da *Implementation Trap*?
5. Valeu a pena o processo de 5 níveis?
6. Alguma decisão fariam diferente?

---

## 🧠 Metodologia Utilizada — Design-First em 5 Níveis

Conforme o artigo de Rahul Garg (disponível em `00_referencias/blog.txt`):

```
NÍVEL 1: CAPACIDADES    → O que o sistema faz?       ✅ Aprovado
NÍVEL 2: COMPONENTES    → Quais são os blocos?        ✅ Aprovado
NÍVEL 3: INTERAÇÕES     → Como se comunicam?          ✅ Aprovado
NÍVEL 4: CONTRATOS      → Quais as interfaces?        ✅ Aprovado
NÍVEL 5: IMPLEMENTAÇÃO  → Tecnologias e diagramas     ✅ Concluído
```

**Regra seguida:** nenhuma tecnologia foi escolhida, nenhum diagrama técnico definitivo foi produzido antes da aprovação do Nível 4 pelo grupo.

---

## 📋 Checklist de Entregáveis

- [x] Evidências do processo Design-First (este AGENT.md + documentos por nível)
- [x] Nome do LLM utilizado: **Claude Sonnet (Anthropic)** via Antigravity IDE
- [x] Diagramas de arquitetura (C4 em 3 níveis + diagramas de sequência Mermaid)
- [x] Decisões arquiteturais documentadas (5 ADRs)
- [x] Comparativo entre proposta anterior e nova
- [ ] **Documento de opiniões do grupo** — escrever manualmente em `03_entregaveis/comparativo_propostas.md`

---

*"O valor do quadro-branco nunca foi o diagrama em si. Era o alinhamento — o entendimento compartilhado que se desenvolve quando duas pessoas pensam juntas sobre um problema, um passo de cada vez."*  
— Rahul Garg
