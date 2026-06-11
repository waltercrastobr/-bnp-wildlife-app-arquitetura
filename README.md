# 🦁 BNP Wildlife App — Arquitetura de Software

> **Disciplina:** Arquitetura de Sistemas | **Professor:** Kiev Gama  
> **Instituição:** UFPE 
> **Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra  
> **Data:** 08/06/2026

---

## 📌 Sobre o Projeto

Este repositório documenta o processo de **design arquitetural** do *Bwindi Nature Park (BNP) Wildlife App* — uma plataforma mobile para rastreamento de vida selvagem, registro de avistamentos e resposta a emergências em um parque natural.

O app serve visitantes, guias de safari, equipe médica e o Oficial de Controle do parque, todos usando um **único app com perfis baseados em role**.

### Funcionalidades principais:
- 🗺️ Mapa ao vivo com avistamentos de animais (atualizado em ≤ 30s)
- 🦁 Registro de avistamentos com controle automático de aglomeração
- 🚨 Botão de pânico com modo degradado offline
- 📡 Feed em tempo real para o Oficial de Controle
- 🏥 Despacho automático de guias e equipe médica por proximidade

---

## 🤖 LLM Utilizado

| Campo | Valor |
|-------|-------|
| **Modelo** | Claude Sonnet 4.6 (Anthropic) |
| **Interface** | Antigravity IDE |
| **Metodologia** | Design-First em 5 níveis progressivos |

O trabalho foi feito em **colaboração ativa** entre o grupo e a IA: a IA apresentava 2–3 opções concretas para cada decisão de design, e o grupo escolhia e aprovava antes de qualquer artefato técnico ser gerado. Nenhuma tecnologia foi escolhida antes que Capacidades, Componentes, Interações e Contratos fossem definidos e aprovados.

---

## 📁 Estrutura do Repositório

```
📦 arquitetura_software/
│
├── 📄 AGENT.md                        ← Índice geral + resumo de todas as decisões
│
├── 📂 00_referencias/                 ← Material de entrada (não editável)
│   ├── desc_atividade.txt             → Enunciado da atividade
│   ├── blog.txt                       → Artigo Design-First (Rahul Garg / Fowler)
│   ├── Designing with AI_...pdf       → Especificação do app BNP
│   └── Proposta de Arquitetura...pdf  → Proposta anterior (referência comparativa)
│
├── 📂 01_design/                      ← Decisões progressivas aprovadas pelo grupo
│   ├── nivel_1_capacidades.md         ✅ Capacidades, escopo, SLOs e modo degradado
│   ├── nivel_2_componentes.md         ✅ 10 componentes com responsabilidades e diagrama C4 textual
│   ├── nivel_3_interacoes.md          ✅ Padrões de comunicação + 4 diagramas de sequência (Mermaid)
│   └── nivel_4_contratos.md           ✅ 9 endpoints REST + 5 eventos de barramento + padrão de erros
│
├── 📂 02_implementacao/               ← Artefatos técnicos finais
│   ├── stack_tecnologica.md           ✅ Stack completa com configurações detalhadas por componente
│   ├── 📂 diagramas/
│   │   ├── diagrama_c4_completo.md    ✅ C4 Context + Container + Component (×2) + Deploy Kubernetes
│   │   └── 📂 fotos/
│   │       ├── c4_context.png         ← C4 Nível 1: atores e sistema
│   │       ├── c4_container.png       ← C4 Nível 2: todos os 10 componentes
│   │       ├── c4_component_emergency.png  ← C4 Nível 3: internals do Emergency Service
│   │       ├── c4_component_sighting.png   ← C4 Nível 3: pipeline de avistamento
│   │       └── deploy_kubernetes.png  ← Visão de deploy com namespaces
│   └── 📂 adrs/                       ← Architectural Decision Records
│       ├── adr_001_event_broker.md    ✅ Apache Kafka vs RabbitMQ
│       ├── adr_002_database_strategy.md  ✅ PostgreSQL+PostGIS + MongoDB
│       ├── adr_003_realtime_protocol.md  ✅ Server-Sent Events vs WebSocket
│       ├── adr_004_mobile_platform.md   ✅ Nativo (Swift+Kotlin) vs Cross-platform
│       └── adr_005_backend_language.md  ✅ Go vs Node.js vs Python
│
└── 📂 03_entregaveis/
    ├── evidencias_atividade.md        ✅ LLM, todos os prompts, 27 decisões, diagramas renderizados
    └── comparativo_propostas.md       ⚠️ Esqueleto pronto — OPINIÕES DO GRUPO PENDENTES
```

---

## 🏗️ Arquitetura Resumida

### Stack Tecnológica Aprovada

| Camada | Tecnologia | Decisão |
|--------|-----------|---------|
| **Barramento de eventos** | Apache Kafka (KRaft) | Retenção de emergências + filas CRITICAL/STANDARD |
| **BD relacional/geoespacial** | PostgreSQL 16 + PostGIS | Queries `ST_DWithin` para crowding |
| **BD de documentos** | MongoDB 7 | Avistamentos, espécies, logs de emergência |
| **Real-time CO** | Server-Sent Events (SSE) | Feed unidirecional com reconexão automática |
| **Mobile** | Swift (iOS) + Kotlin (Android) | GPS background confiável + botão de pânico |
| **Backend** | Go (Golang) | Goroutines leves + GC de baixa latência |
| **Orquestração** | Kubernetes | Emergency Service isolado com `PriorityClass` |

### Decisões-Chave

```
27 decisões tomadas ao longo de 5 níveis:

Nível 1 — Capacidades:  geo-fenced · SLO ≤30s · offline queue · app único · greenfield
Nível 2 — Componentes:  10 serviços · Location Tracker dedicado · Emergency isolado
Nível 3 — Interações:   híbrido sínc/assínc · batch GPS 30s · polling 15s · SSE para CO
Nível 4 — Contratos:    JWT auto-suficiente · speciesId · trajetória em lote · deduplicação
Nível 5 — Tecnologia:   Kafka · PostgreSQL+MongoDB · SSE · Swift+Kotlin · Go
```

---

## 📋 Checklist de Entregáveis

- [x] Nome do LLM utilizado
- [x] Evidências do processo Design-First (prompts literais)
- [x] Diagramas de arquitetura (C4 em 3 níveis + sequência)
- [x] Decisões arquiteturais documentadas (5 ADRs)
- [x] Comparativo entre proposta anterior e nova proposta
- [ ] **Opiniões individuais do grupo** → `03_entregaveis/comparativo_propostas.md`

---

## 🔗 Referência Metodológica

Baseado no artigo **"Design-First Collaboration with AI"** de Rahul Garg  
Disponível em: `00_referencias/blog.txt`  
Princípio central: *"Nenhuma tecnologia é escolhida antes que capacidades, componentes, interações e contratos sejam aprovados pelo grupo."*
