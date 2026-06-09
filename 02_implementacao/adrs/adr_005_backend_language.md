# ADR-005: Go (Golang) para os Microserviços Backend

**Status:** Aceito  
**Data:** 08/06/2026  
**Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra

---

## Contexto

O backend do BNP é composto por 8 microserviços com perfis de carga bem distintos:

- **Location Tracker Service:** recebe centenas de lotes de posições simultâneos a cada 30s — workload de alta concorrência de I/O
- **Crowding Prevention Engine:** executa queries geoespaciais síncronas no PostgreSQL para cada avistamento — workload misto CPU+I/O
- **Emergency & Control Service:** latência mínima é critério de segurança — zero tolerância a garbage collection pauses
- **Sighting Service / Notification Service:** workload de I/O assíncrono orientado a eventos (Kafka consumer/producer)
- **Map Service:** serve tiles estáticas — workload de I/O puro, alta taxa de requests

Todos os serviços precisam de:
- Consumo de memória baixo (múltiplos serviços em Kubernetes no mesmo cluster)
- Startup rápido (escalabilidade horizontal ágil em picos)
- Concorrência eficiente sem overhead de threads pesadas

## Decisão

Adotamos **Go 1.22+** para todos os microserviços backend.

- **Router HTTP:** `chi` (leve, idiomático em Go, suporte nativo a middleware)
- **Kafka:** `confluent-kafka-go` (bindings oficiais para librdkafka)
- **PostgreSQL:** `pgx/v5` com `pgxpool` para connection pooling
- **MongoDB:** `go.mongodb.org/mongo-driver` oficial
- **SSE:** implementado com `net/http` stdlib — `flusher.Flush()` em loop de goroutine dedicada por conexão

O Emergency & Control Service é deployado com `GOMAXPROCS` fixo para garantir que goroutines de emergência não compitam com outros serviços no mesmo nó.

## Consequências

**Positivas:**
- Goroutines são extremamente leves (~2KB vs ~1MB de uma thread OS) — o Location Tracker pode processar centenas de lotes simultâneos com goroutines individuais sem overhead
- Go compila para binário único sem runtime externo — imagens Docker mínimas (~10-20MB), startup em milissegundos
- Garbage collector com pausas de latência baixa (<1ms para a maioria dos casos) — crítico para o Emergency Service
- Concorrência explícita com channels e goroutines — o modelo de comunicação do Go espelha naturalmente o design de eventos do sistema
- `net/http` nativo suporta HTTP/2 — SSE do CONTROL_OFFICER funciona com multiplexing sem configuração adicional

**Negativas (trade-offs aceitos):**
- Go tem um ecossistema menor que Node.js/Python para alguns domínios
- A ausência de generics histórica (Go 1.18+ já tem) ainda faz alguns padrões mais verbosos
- Dois apps nativos (iOS/Android) + Go backend = três linguagens/ecossistemas distintos para o time

## Alternativas Consideradas

**Node.js com TypeScript:** Excelente para I/O assíncrono, mesmo ecossistema do frontend. Descartado porque o event loop single-threaded do Node.js tem comportamento imprevisível sob carga CPU-bound (como queries geoespaciais complexas do Crowding Engine), e as pausas do V8 GC podem impactar o SLA do Emergency Service.

**Python com FastAPI:** Familiar e rápido para prototipagem. Descartado pela limitação do GIL (Global Interpreter Lock) em operações CPU-bound concorrentes e pela performance inferior sob alta concorrência de conexões simultâneas.
