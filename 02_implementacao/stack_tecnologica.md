# Stack Tecnológica — BNP Wildlife App ✅ APROVADO

> **Status:** Aprovado pelo grupo  
> **Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra  
> **Data:** 08/06/2026  
> **Baseado em:** Decisões dos Níveis 1 a 4

---

## Visão Geral da Stack

| Camada | Tecnologia Escolhida | Decisão de Origem |
|--------|----------------------|-------------------|
| **Barramento de Eventos** | Apache Kafka | D5.1 |
| **Banco Relacional** | PostgreSQL + PostGIS | D5.2 + D5.3 |
| **Banco de Documentos** | MongoDB | D5.2 |
| **Canal em Tempo Real (CO Feed)** | Server-Sent Events (SSE) | D5.4 |
| **App Mobile** | Swift (iOS) + Kotlin (Android) — nativo | D5.5 |
| **Microserviços Backend** | Go (Golang) | D5.6 |
| **API Gateway** | Kong ou custom Go HTTP proxy | D5.6 |
| **Contêinerização** | Docker + Kubernetes | Consequência de D2.5 (isolamento do Emergency Service) |
| **Mapa base** | Tiles próprios servidos via Map Service (Go) | D2.3 |

---

## Detalhamento por Componente

### 🟠 Apache Kafka — Barramento de Eventos
- **Versão:** Kafka 3.x com KRaft (sem Zookeeper)
- **Tópicos configurados:**
  - `bnp.sightings.submitted` — partições: 6, retenção: 7 dias
  - `bnp.sightings.approved` — partições: 6, retenção: 7 dias
  - `bnp.sightings.suppressed` — partições: 3, retenção: 3 dias
  - `bnp.emergency.raised` — partições: 3, retenção: 30 dias, **prioridade máxima**
  - `bnp.emergency.dispatched` — partições: 3, retenção: 30 dias
- **Grupos de consumidores:**
  - `crowding-engine-group` → consome `bnp.sightings.submitted`
  - `notification-service-group` → consome aprovados e suprimidos
  - `emergency-control-group` → consome `bnp.emergency.raised` com **max.poll.records=1** (processa um por vez, sem atraso por lote)

---

### 🐘 PostgreSQL 16 + PostGIS — Banco Relacional e Geoespacial
- **Responsabilidade:** dados de usuários, veículos, autenticação, ecossistemas do parque e **posições ao vivo dos veículos** (Location Tracker)
- **Extensões:** PostGIS 3.x
- **Tabelas principais:**
  - `users` — id, name, email, phone, role, vehicle_plate
  - `vehicles` — id, plate, owner_id
  - `park_areas` — id, name, geometry (POLYGON), ecosystem_type
  - `species` — id, common_name, scientific_name, category
  - `vehicle_locations` — vehicle_id, location (POINT), captured_at, updated_at
  - `staff_locations` — staff_id, role, location (POINT), updated_at
- **Queries críticas:**
  - Crowding: `SELECT COUNT(*) FROM vehicle_locations WHERE ST_DWithin(location, ST_Point(lng, lat)::geography, radius_meters)`
  - Nearest staff: `SELECT * FROM staff_locations ORDER BY location <-> ST_Point(lng, lat) LIMIT 5`

---

### 🍃 MongoDB 7 — Banco de Documentos
- **Responsabilidade:** histórico de avistamentos, catálogo de espécies com metadados ricos (fotos, descrições, comportamentos), logs de emergências
- **Collections:**
  - `sightings` — { sightingId, speciesId, location: {lat, lng}, timestamp, reporterId, status, vehicleCountAtApproval }
  - `emergencies` — { emergencyId, visitorId, location, timestamp, assignedTeams, status, timeline: [...] }
- **Índices:**
  - `sightings`: índice geoespacial 2dsphere em `location`, índice em `timestamp` (para polling eficiente)
  - `emergencies`: índice em `emergencyId` (deduplicação do modo offline)

---

### 📡 Server-Sent Events (SSE) — CONTROL_OFFICER Feed
- **Endpoint:** `GET /co-feed` (autenticado, role: CONTROL_OFFICER)
- **Implementado em:** Go com `net/http` nativo (sem biblioteca externa)
- **Eventos emitidos:**
  - `vehicle.position.updated` — posição de todos os veículos ativos (a cada 30s, em lote)
  - `sighting.published` — novo avistamento aprovado
  - `emergency.raised` — alerta de pânico com localização
  - `emergency.dispatched` — confirmação de despacho com equipes
  - `emergency.resolved` — encerramento de emergência
- **Reconexão:** SSE tem reconexão automática nativa via campo `retry:` no protocolo

---

### 📱 Swift (iOS) + Kotlin (Android) — Apps Nativos
- **Motivação:** GPS em background, geofencing, notificações push críticas e captura offline de emergência exigem acesso total às APIs nativas de cada plataforma.
- **iOS (Swift):**
  - CLLocationManager — rastreamento GPS contínuo em background
  - CoreLocation geofencing — detecta entrada/saída do perímetro do BNP
  - APNs (Apple Push Notification Service) — push para notificações de avistamento
  - URLSession — HTTP para API calls e SSE
  - Core Data — persistência local para fila offline de emergências
- **Android (Kotlin):**
  - FusedLocationProvider — GPS otimizado para bateria
  - Geofencing API — perímetro do parque
  - FCM (Firebase Cloud Messaging) — push notifications
  - OkHttp — HTTP e SSE
  - Room Database — persistência local para fila offline de emergências
- **Lógica compartilhada:** Os dois apps implementam os mesmos contratos definidos no Nível 4. Qualquer divergência de comportamento entre plataformas é um bug.

---

### 🐹 Go (Golang) — Microserviços Backend
- **Versão:** Go 1.22+
- **Framework HTTP:** `net/http` (stdlib) + `chi` router para simplicidade
- **Kafka client:** `confluent-kafka-go`
- **PostgreSQL client:** `pgx/v5` com pool de conexões
- **MongoDB client:** `mongo-driver` oficial
- **Serviços em Go:**
  - Auth & User Service
  - Park Information Service
  - Location Tracker Service ← beneficia-se especialmente de goroutines para processar lotes de posições concorrentemente
  - Sighting Service
  - Crowding Prevention Engine
  - Map Service (serve tiles via HTTP)
  - Notification Service
  - Emergency & Control Service ← deploy em pod Kubernetes isolado com `resources.requests` e `resources.limits` dedicados

---

### ☸️ Kubernetes — Orquestração de Contêineres
- Cada microserviço em Go é um contêiner Docker separado
- **Emergency & Control Service:** pod com `PriorityClass: high-priority` e `resources.limits` fixos (CPU e memória garantidos, não compartilhados)
- **Kafka:** Kafka Cluster via Helm chart (3 brokers para produção)
- **PostgreSQL:** StatefulSet com volume persistente
- **MongoDB:** StatefulSet com replica set de 3 nós

---

## Diagrama de Tecnologias por Camada

```
┌──────────────────────────────────────────────────────────┐
│               CLIENTES                                    │
│  iOS App (Swift)         Android App (Kotlin)            │
│  • CoreLocation          • FusedLocationProvider         │
│  • APNs                  • FCM                           │
│  • Core Data (offline)   • Room DB (offline)             │
└────────────────────────┬─────────────────────────────────┘
                         │ HTTPS
┌────────────────────────▼─────────────────────────────────┐
│              API GATEWAY (Go)                             │
│  • Validação de JWT    • Rate Limiting                   │
│  • Roteamento          • SSE endpoint /co-feed           │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│           MICROSERVIÇOS (Go)                             │
│                                                          │
│  Auth & User  │  Park Info  │  Location Tracker          │
│  Sighting     │  Crowding   │  Map Service               │
│  Notification │  Emergency & Control ⚠️[ISOLADO]         │
└──┬────────────┬─────────────────────┬────────────────────┘
   │            │                     │
┌──▼──┐    ┌───▼────────┐     ┌──────▼──────────────┐
│Kafka│    │ PostgreSQL │     │      MongoDB         │
│     │    │ + PostGIS  │     │                      │
│CRIT.│    │ users      │     │ sightings (histórico)│
│STD  │    │ vehicles   │     │ emergencies (log)    │
│     │    │ park_areas │     │ species (catálogo)   │
│     │    │ locations  │     │                      │
└─────┘    └────────────┘     └──────────────────────┘
```
