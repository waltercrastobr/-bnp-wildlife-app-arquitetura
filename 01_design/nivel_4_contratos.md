# Nível 4 — Contratos ✅ APROVADO

> **Status:** Aprovado pelo grupo  
> **Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra  
> **Data:** 08/06/2026  
> **Baseado em:** Nível 1 + Nível 2 + Nível 3

---

## Decisões de Contrato

| Decisão | Escolha |
|---------|---------|
| D4.1 — Token | Token auto-suficiente com `userId`, `role`, `vehiclePlate` e `expiresAt` embutidos |
| D4.2 — Tipo de animal | `speciesId` (UUID) referenciando catálogo carregado do backend ao entrar no parque |
| D4.3 — Localização em lote | Array de todas as posições do período (trajetória dos últimos 30s) |
| D4.4 — Resposta de emergência | Síncrono até o despacho: resposta inclui equipes já selecionadas e ETA |
| D4.5 — Polling do mapa | Retorna todos os avistamentos do parque desde o último timestamp; app filtra localmente |

---

## 1. Token de Autenticação

Emitido pelo **Auth & User Service** após login bem-sucedido. Validado pelo **API Gateway** em toda requisição via header `Authorization: Bearer <token>`.

```json
{
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "role": "VISITOR",
  "vehiclePlate": "ABC-1234",
  "phoneNumber": "+258841234567",
  "expiresAt": "2026-06-08T23:59:59Z",
  "issuedAt": "2026-06-08T18:00:00Z"
}
```

**Roles válidos:**
```
"VISITOR"          → acesso a: park-info, sightings, subscriptions, locations, emergency
"GUIDE"            → acesso a: tudo do VISITOR + recebimento de alertas de emergência
"MEDICAL_STAFF"    → acesso a: recebimento de alertas de emergência
"CONTROL_OFFICER"  → acesso a: tudo + co-feed, emergency dispatch
```

**Regras de negócio:**
- `vehiclePlate` é `null` para roles `GUIDE`, `MEDICAL_STAFF` e `CONTROL_OFFICER`
- Token expira em 8 horas (dentro da janela de uma visita ao parque)
- Token revogado imediatamente em logout — API Gateway mantém lista negra local com TTL curto

---

## 2. Endpoints da API

### 2.1 — Autenticação

#### `POST /auth/register`
Cadastro de novo usuário (visitante ou staff criado internamente).

**Request:**
```json
{
  "name": "Walter Monteiro",
  "email": "walter@email.com",
  "password": "string (mín. 8 chars)",
  "phoneNumber": "+258841234567",
  "vehiclePlate": "ABC-1234",
  "role": "VISITOR"
}
```
**Response `201 Created`:**
```json
{
  "userId": "uuid",
  "message": "Conta criada. Faça login para obter seu token."
}
```

---

#### `POST /auth/login`
**Request:**
```json
{ "email": "walter@email.com", "password": "string" }
```
**Response `200 OK`:**
```json
{
  "token": "<token-string>",
  "userId": "uuid",
  "role": "VISITOR",
  "expiresAt": "ISO8601"
}
```

---

### 2.2 — Catálogo de Espécies

#### `GET /species`
Carregado pelo app ao entrar no parque (geofence ativo). Resposta é cacheável localmente.

**Response `200 OK`:**
```json
{
  "species": [
    {
      "speciesId": "uuid",
      "commonName": "Leão",
      "scientificName": "Panthera leo",
      "category": "BIG_FIVE",
      "iconUrl": "/species/icons/lion.png",
      "description": "Maior felino da África..."
    },
    {
      "speciesId": "uuid",
      "commonName": "Elefante Africano",
      "scientificName": "Loxodonta africana",
      "category": "BIG_FIVE",
      "iconUrl": "/species/icons/elephant.png",
      "description": "Maior animal terrestre..."
    }
  ]
}
```

---

### 2.3 — Informações do Parque

#### `GET /park-info?lat={lat}&lng={lng}`
Retorna dados do ecossistema baseado nas coordenadas do visitante.

**Response `200 OK`:**
```json
{
  "areaName": "Savana Norte",
  "ecosystemType": "SAVANNA",
  "description": "Região de transição com vegetação de acácia...",
  "restrictions": ["MAX_SPEED_20KMH", "NO_STOPPING"],
  "commonSpecies": ["uuid-leao", "uuid-zebra", "uuid-gnu"],
  "coordinates": {
    "boundingBox": {
      "northEast": { "lat": -1.200, "lng": 36.850 },
      "southWest": { "lat": -1.300, "lng": 36.750 }
    }
  }
}
```

---

### 2.4 — Avistamentos

#### `POST /sightings`
Visitante registra avistamento de animal.

**Request:**
```json
{
  "speciesId": "uuid-da-especie",
  "location": {
    "lat": -1.2673,
    "lng": 36.7983
  },
  "timestamp": "2026-06-08T18:34:00Z",
  "notes": "Filhote junto da mãe (opcional)"
}
```

**Response `202 Accepted`:**
```json
{
  "sightingId": "uuid",
  "status": "PENDING",
  "message": "Avistamento recebido. Será publicado após verificação de aglomeração."
}
```

**Possíveis status no ciclo de vida de um avistamento:**
```
PENDING    → recebido, aguardando verificação de crowding
APPROVED   → publicado no mapa e notificações enviadas
SUPPRESSED → bloqueado por crowding — área com muitos veículos
```

---

#### `GET /sightings?since={ISO8601}`
Polling do mapa. Retorna todos os avistamentos aprovados desde o timestamp informado.

**Response `200 OK`:**
```json
{
  "sightings": [
    {
      "sightingId": "uuid",
      "speciesId": "uuid-da-especie",
      "speciesName": "Leão",
      "location": { "lat": -1.2673, "lng": 36.7983 },
      "timestamp": "2026-06-08T18:34:00Z",
      "reportedBy": "VISITOR"
    }
  ],
  "lastUpdatedAt": "2026-06-08T18:34:30Z"
}
```
> O app armazena `lastUpdatedAt` e usa como valor de `since` na próxima chamada de polling.

---

### 2.5 — Assinaturas de Notificação

#### `POST /subscriptions`
Visitante se inscreve para receber notificações de uma espécie específica.

**Request:**
```json
{ "speciesId": "uuid-da-especie" }
```
**Response `201 Created`:**
```json
{ "subscriptionId": "uuid", "speciesId": "uuid", "status": "ACTIVE" }
```

#### `DELETE /subscriptions/{subscriptionId}`
Cancela assinatura.
**Response `204 No Content`**

---

### 2.6 — Localização em Lote

#### `POST /locations/batch`
Enviado pelo app em background a cada 30 segundos para todos os roles ativos no parque.

**Request:**
```json
{
  "vehicleId": "uuid-do-veiculo-ou-userId-para-staff",
  "role": "VISITOR",
  "positions": [
    { "lat": -1.2670, "lng": 36.7980, "capturedAt": "2026-06-08T18:33:30Z" },
    { "lat": -1.2671, "lng": 36.7981, "capturedAt": "2026-06-08T18:33:40Z" },
    { "lat": -1.2673, "lng": 36.7983, "capturedAt": "2026-06-08T18:34:00Z" }
  ]
}
```
> O Location Tracker extrai a posição com `capturedAt` mais recente como posição atual e persiste a trajetória.

**Response `204 No Content`**

---

### 2.7 — Emergência

#### `POST /emergency`
Botão de pânico. Chamada síncrona — servidor aguarda seleção de equipes antes de responder.

**Request:**
```json
{
  "emergencyId": "uuid-gerado-localmente-pelo-app",
  "location": { "lat": -1.2673, "lng": 36.7983 },
  "timestamp": "2026-06-08T18:34:00Z",
  "vehiclePlate": "ABC-1234"
}
```
> `emergencyId` é gerado pelo app para suporte à deduplicação no modo degradado (offline queue).

**Response `200 OK`:**
```json
{
  "emergencyId": "uuid",
  "status": "DISPATCHED",
  "assignedTeams": [
    {
      "memberId": "uuid",
      "name": "Guia Carlos Mwangi",
      "role": "GUIDE",
      "distanceMeters": 1200,
      "etaMinutes": 4
    },
    {
      "memberId": "uuid",
      "name": "Dra. Amina Osei",
      "role": "MEDICAL_STAFF",
      "distanceMeters": 3500,
      "etaMinutes": 9
    }
  ],
  "controlOfficer": "José Silva",
  "message": "Equipes de socorro despachadas. Mantenha sua localização."
}
```

---

#### `PUT /emergency/{emergencyId}/dispatch`
CONTROL_OFFICER confirma ou ajusta as equipes despachadas.

**Request:**
```json
{
  "confirmedTeams": ["uuid-guia", "uuid-medico"],
  "additionalNotes": "Levar maca de emergência"
}
```
**Response `200 OK`:**
```json
{
  "emergencyId": "uuid",
  "status": "CONFIRMED",
  "updatedAt": "ISO8601"
}
```

---

### 2.8 — Tiles de Mapa

#### `GET /map/tiles/{z}/{x}/{y}`
Padrão TMS (Tile Map Service). Retorna tiles cartográficas do parque.

**Response `200 OK`:** imagem binária (PNG/WebP) da tile  
**Response `404 Not Found`:** tile fora do perímetro do parque

#### `GET /map/manifest`
Retorna o manifesto de tiles disponíveis para download offline ao entrar no parque.

**Response `200 OK`:**
```json
{
  "boundingBox": { "northEast": {...}, "southWest": {...} },
  "zoomLevels": [10, 11, 12, 13, 14],
  "totalTiles": 847,
  "estimatedSizeBytes": 25400000,
  "version": "2026-06-08"
}
```

---

## 3. Esquemas de Eventos do Barramento

### `sighting.submitted` — Fila: STANDARD

**Produtor:** Sighting Service  
**Consumidor:** Crowding Prevention Engine

```json
{
  "eventId": "uuid",
  "eventType": "sighting.submitted",
  "occurredAt": "ISO8601",
  "payload": {
    "sightingId": "uuid",
    "speciesId": "uuid",
    "location": { "lat": number, "lng": number },
    "timestamp": "ISO8601",
    "reporterVehicleId": "uuid"
  }
}
```

---

### `sighting.approved` — Fila: STANDARD

**Produtor:** Crowding Prevention Engine  
**Consumidores:** Notification Service

```json
{
  "eventId": "uuid",
  "eventType": "sighting.approved",
  "occurredAt": "ISO8601",
  "payload": {
    "sightingId": "uuid",
    "speciesId": "uuid",
    "location": { "lat": number, "lng": number },
    "timestamp": "ISO8601",
    "vehicleCountAtApproval": 2
  }
}
```

---

### `sighting.suppressed` — Fila: STANDARD

**Produtor:** Crowding Prevention Engine  
**Consumidor:** Notification Service (notifica remetente)

```json
{
  "eventId": "uuid",
  "eventType": "sighting.suppressed",
  "occurredAt": "ISO8601",
  "payload": {
    "sightingId": "uuid",
    "reason": "CROWDING_THRESHOLD_EXCEEDED",
    "vehicleCountAtCheck": 8,
    "threshold": 7,
    "reporterVehicleId": "uuid"
  }
}
```

---

### `emergency.raised` — Fila: **CRITICAL**

**Produtor:** API Gateway (ao receber POST /emergency)  
**Consumidor:** Emergency & Control Service (consumidor prioritário)

```json
{
  "eventId": "uuid",
  "eventType": "emergency.raised",
  "occurredAt": "ISO8601",
  "priority": "CRITICAL",
  "payload": {
    "emergencyId": "uuid",
    "visitorId": "uuid",
    "vehiclePlate": "ABC-1234",
    "location": { "lat": number, "lng": number },
    "clientTimestamp": "ISO8601"
  }
}
```

---

### `emergency.dispatched` — Fila: **CRITICAL**

**Produtor:** Emergency & Control Service  
**Consumidores:** Notification Service [CRITICAL], CO Feed

```json
{
  "eventId": "uuid",
  "eventType": "emergency.dispatched",
  "occurredAt": "ISO8601",
  "priority": "CRITICAL",
  "payload": {
    "emergencyId": "uuid",
    "location": { "lat": number, "lng": number },
    "assignedTeams": [
      { "memberId": "uuid", "name": "string", "role": "GUIDE", "etaMinutes": number }
    ],
    "controlOfficerId": "uuid"
  }
}
```

---

## 4. Erros — Padrão Comum de Resposta

Todos os endpoints de erro seguem o mesmo esquema:

```json
{
  "errorCode": "SIGHTING_AREA_SUPPRESSED",
  "message": "Avistamento bloqueado: número máximo de veículos na área atingido.",
  "details": { },
  "timestamp": "ISO8601",
  "requestId": "uuid"
}
```

**Códigos de erro relevantes:**

| Código | HTTP | Situação |
|--------|------|----------|
| `UNAUTHORIZED` | 401 | Token ausente ou inválido |
| `FORBIDDEN_ROLE` | 403 | Role sem permissão para o endpoint |
| `NOT_IN_PARK` | 403 | Funcionalidade requer estar dentro do perímetro do parque |
| `SIGHTING_AREA_SUPPRESSED` | 200 | Avistamento recebido mas suprimido por crowding |
| `EMERGENCY_DUPLICATE` | 200 | emergencyId já registrado (deduplicação do modo offline) |
| `SPECIES_NOT_FOUND` | 404 | speciesId não encontrado no catálogo |
| `LOCATION_REQUIRED` | 422 | GPS não disponível para operação que exige localização |
