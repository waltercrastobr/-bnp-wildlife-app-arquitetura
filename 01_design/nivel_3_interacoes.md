# Nível 3 — Interações ✅ APROVADO

> **Status:** Aprovado pelo grupo  
> **Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra  
> **Data:** 08/06/2026  
> **Baseado em:** Nível 1 (Capacidades) + Nível 2 (Componentes)

---

## Decisões de Interação

| Decisão | Escolha |
|---------|---------|
| D3.1 — Padrão geral | **Híbrido:** síncrono para leituras e operações que exigem resposta imediata; assíncrono via barramento de eventos para processamento em pipeline |
| D3.2 — Envio de localização | **Lote a cada 30s:** app acumula posições localmente e envia pacote para o Location Tracker |
| D3.3 — Atualização do mapa | **Polling:** app consulta o servidor a cada intervalo (alvo: ~15s) para buscar avistamentos novos, dentro do SLO de 30s |
| D3.4 — Botão de pânico | **Barramento de eventos com fila CRITICAL:** alerta entra no bus como `emergency.raised [CRITICAL]`; Emergency & Control Service é consumidor prioritário dessa fila |
| D3.5 — Feed do CONTROL_OFFICER | **Canal dedicado:** conexão persistente exclusiva para o oficial, com feed completo do parque (todos os veículos, avistamentos, emergências ativas) |

---

## Padrão Híbrido — Regras Gerais

```
SÍNCRONO (chamada direta, aguarda resposta):
  ✅ Leituras: buscar info de ecossistema, tiles de mapa, dados do perfil
  ✅ Registro inicial: submeter avistamento (retorna 202 Accepted imediatamente)
  ✅ Consultas de estado: Location Tracker consultado pelo Crowding Engine
  ✅ Rota de emergência: Mobile App → API Gateway → Event Bus [CRITICAL]

ASSÍNCRONO (barramento de eventos, processamento em background):
  ✅ Pipeline de avistamento: Sighting → Crowding → Notification
  ✅ Alertas de emergência: Emergency [CRITICAL] → Dispatcher → Notificação das equipes
  ✅ Broadcast de mapa: aprovação de avistamento → atualização disponível para polling

CONEXÃO PERSISTENTE (canal aberto, servidor empurra dados):
  ✅ Exclusivo para CONTROL_OFFICER: feed completo do parque em tempo real
```

---

## Diagrama de Sequência — Fluxo 1: Informações Gerais

> Visitante entra em uma área do parque e o app exibe informações do ecossistema local.

```mermaid
sequenceDiagram
    actor V as Visitante
    participant App as Mobile App
    participant GW as API Gateway
    participant PIS as Park Information Service
    participant DB as Data Layer

    V->>App: Abre o app / move-se para nova área
    App->>App: Detecta coordenadas GPS atuais
    App->>GW: GET /park-info?lat=X&lng=Y [sync]
    GW->>GW: Valida token JWT (role: VISITOR)
    GW->>PIS: GET /ecosystem?lat=X&lng=Y [sync]
    PIS->>DB: Consulta área geográfica por coordenadas [sync]
    DB-->>PIS: Dados do ecossistema (fauna, flora, restrições)
    PIS-->>GW: 200 OK { ecosystemData }
    GW-->>App: 200 OK { ecosystemData }
    App->>V: Exibe informações da área atual
```

---

## Diagrama de Sequência — Fluxo 2: Avistamento + Crowding + Notificação

> Visitante registra avistamento. Sistema verifica aglomeração e decide publicar ou suprimir.

```mermaid
sequenceDiagram
    actor V as Visitante
    participant App as Mobile App
    participant GW as API Gateway
    participant SS as Sighting Service
    participant Bus as Barramento de Eventos
    participant CPE as Crowding Prevention Engine
    participant LT as Location Tracker Service
    participant NS as Notification Service
    participant Apps as Apps dos Inscritos

    V->>App: Registra avistamento (animal + GPS)
    App->>GW: POST /sightings [sync]
    GW->>GW: Valida token JWT
    GW->>SS: POST /sightings [sync]
    SS->>SS: Valida e persiste avistamento
    SS-->>GW: 202 Accepted { sightingId, status: "pending" }
    GW-->>App: 202 Accepted — avistamento recebido
    App->>V: Feedback: "Avistamento registrado!"

    Note over SS,Bus: Processamento assíncrono começa aqui

    SS->>Bus: Publica evento: sighting.submitted { sightingId, animal, lat, lng }

    Bus->>CPE: Consome: sighting.submitted

    CPE->>LT: GET /vehicles/count?lat=X&lng=Y&radius=R [sync]
    LT-->>CPE: { count: N, threshold: T }

    alt count < threshold (área não saturada)
        CPE->>Bus: Publica: sighting.approved { sightingId }
        Bus->>NS: Consome: sighting.approved [fila STANDARD]
        NS->>NS: Busca inscritos no tipo de animal
        NS->>Apps: Push notification para inscritos
        Note over Apps: Apps em polling receberão o avistamento<br/>na próxima consulta (≤ 15s intervalo de polling)
    else count >= threshold (área saturada)
        CPE->>Bus: Publica: sighting.suppressed { sightingId, reason: "crowding" }
        Bus->>NS: Consome: sighting.suppressed [fila STANDARD]
        NS->>App: Notifica remetente: "Área com muitos visitantes — avistamento não publicado"
    end
```

> **SLO de 30s:** o pipeline completo (POST → evento aprovado → push) deve completar em ≤ 30 segundos.  
> **Polling do mapa:** visitantes consultam novos avistamentos a cada ~15s. Um avistamento aprovado estará disponível na próxima consulta após a aprovação.

---

## Diagrama de Sequência — Fluxo 3: Botão de Pânico → Despacho → Coordenação

> Visitante aciona emergência. Sistema identifica equipes mais próximas. CONTROL_OFFICER coordena resposta.

```mermaid
sequenceDiagram
    actor V as Visitante
    participant App as Mobile App
    participant GW as API Gateway
    participant Bus as Barramento de Eventos
    participant ECS as Emergency & Control Service
    participant LT as Location Tracker Service
    participant NS as Notification Service
    participant GuideApp as App do Guia/Médico
    actor CO as CONTROL_OFFICER
    participant COFeed as Canal Dedicado (CO Feed)

    V->>App: Pressiona botão de pânico
    App->>App: Captura GPS atual + timestamp

    alt COM conectividade
        App->>GW: POST /emergency [sync, alta prioridade]
        GW->>GW: Valida token JWT
        GW->>Bus: Publica: emergency.raised [fila CRITICAL] { visitorId, lat, lng, timestamp, vehiclePlate }
        GW-->>App: 200 OK { emergencyId, status: "dispatching" }
        App->>V: "Alerta enviado. Aguarde socorro."
    else SEM conectividade (modo degradado — D3 do Nível 1)
        App->>App: Salva localmente { emergencyId, lat, lng, timestamp, vehiclePlate }
        App->>V: "Alerta registrado. Enviando quando sinal retornar..."
        App-->>GW: (reenvia automaticamente ao reconectar)
    end

    Note over Bus,ECS: Emergency & Control Service é consumidor<br/>PRIORITÁRIO da fila CRITICAL

    Bus->>ECS: Consome: emergency.raised [CRITICAL — processado antes de qualquer outra fila]
    ECS->>LT: GET /staff/nearby?lat=X&lng=Y [sync]
    LT-->>ECS: { guides: [...], medicalStaff: [...] } (ordenados por proximidade)

    ECS->>NS: Despacha alerta [fila CRITICAL] para guias e médicos selecionados
    NS->>GuideApp: Push CRÍTICO: "EMERGÊNCIA — Visitante em perigo em [coords]. Dirija-se ao local."

    ECS->>COFeed: Emite evento ao canal dedicado do CONTROL_OFFICER
    COFeed->>CO: Alerta em tempo real no painel: { emergencyId, location, assignedTeams, status }

    CO->>App: Confirma/ajusta equipes despachadas via painel
    App->>GW: PUT /emergency/{id}/dispatch [sync]
    GW->>ECS: PUT /emergency/{id}/dispatch { confirmedTeams }
    ECS->>ECS: Registra despacho confirmado
    ECS->>COFeed: Atualiza status no painel do CONTROL_OFFICER
```

---

## Diagrama de Sequência — Fluxo 4: Envio de Localização em Lote (Background)

> O app acumula posições e envia em lote a cada 30 segundos para o Location Tracker.

```mermaid
sequenceDiagram
    participant App as Mobile App (background)
    participant GW as API Gateway
    participant LT as Location Tracker Service
    participant DB as Data Layer (Cache de Posições)

    loop A cada 30 segundos (enquanto in-park)
        App->>App: Coleta posições acumuladas no período
        App->>GW: POST /locations/batch [sync] { vehicleId, positions: [...] }
        GW->>LT: POST /locations/batch
        LT->>LT: Extrai posição mais recente de cada veículo
        LT->>DB: Upsert posição atual do veículo (cache em memória)
        LT-->>GW: 204 No Content
        GW-->>App: 204 — lote recebido
    end
```

---

## Eventos do Barramento — Inventário Preliminar

| Evento | Fila | Produtor | Consumidor(es) |
|--------|------|----------|----------------|
| `sighting.submitted` | STANDARD | Sighting Service | Crowding Prevention Engine |
| `sighting.approved` | STANDARD | Crowding Prevention Engine | Notification Service |
| `sighting.suppressed` | STANDARD | Crowding Prevention Engine | Notification Service |
| `emergency.raised` | **CRITICAL** | API Gateway | Emergency & Control Service |
| `emergency.dispatched` | **CRITICAL** | Emergency & Control Service | Notification Service, CO Feed |

---

## Implicações de Design Identificadas

1. **SLO de 30s é apertado para Polling:** Com polling a cada 15s e pipeline assíncrono, o caminho crítico é: `POST sighting → evento → crowding check → aprovação → disponível no polling`. Esses passos precisam completar em menos de 15s para garantir que a próxima janela de polling já traga o avistamento aprovado.

2. **Location Tracker como componente de leitura crítica:** Tanto o Crowding Prevention Engine quanto o Emergency & Control Service fazem consultas síncronas ao Location Tracker. Ele é um ponto de dependência crítica — precisa ser altamente disponível e de baixa latência.

3. **Canal dedicado do CONTROL_OFFICER:** É o único caso de conexão persistente servidor→cliente no sistema. Deve ser tolerante a reconexões (o CONTROL_OFFICER pode perder conexão e retomar sem perder o estado do parque).

4. **Deduplicação de emergências:** No modo degradado (offline queue), o mesmo alerta pode ser enviado mais de uma vez ao reconectar. O Emergency & Control Service deve detectar e ignorar duplicatas pelo `emergencyId` gerado localmente no app.
