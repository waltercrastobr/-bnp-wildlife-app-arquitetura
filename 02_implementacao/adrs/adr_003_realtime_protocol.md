# ADR-003: Server-Sent Events (SSE) para o Feed do CONTROL_OFFICER

**Status:** Aceito  
**Data:** 08/06/2026  
**Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra

---

## Contexto

O CONTROL_OFFICER precisa de um feed em tempo real com o estado completo do parque: posições de todos os veículos, novos avistamentos, alertas de emergência ativos e atualizações de despacho. Esse feed é predominantemente unidirecional — **servidor empurra dados para o cliente**.

As ações do oficial (confirmar despacho, ajustar equipes) já foram mapeadas como chamadas REST no Nível 4 (`PUT /emergency/{id}/dispatch`). Portanto, não existe necessidade de comunicação bidirecional pelo canal de feed.

## Decisão

Adotamos **Server-Sent Events (SSE)** via `GET /co-feed` para o canal dedicado do CONTROL_OFFICER.

SSE é implementado nativamente em Go com `net/http` — o servidor escreve eventos no formato:
```
event: emergency.raised
data: {"emergencyId":"uuid","location":{"lat":-1.26,"lng":36.79},...}

event: vehicle.position.updated
data: {"vehicleId":"uuid","lat":-1.27,"lng":36.80,...}
```

O cliente (app nativo iOS/Android) conecta uma única vez e recebe o stream de eventos. Reconexão é automática via o campo `retry:` do protocolo SSE, com o header `Last-Event-ID` para retomar sem perder eventos.

## Consequências

**Positivas:**
- Implementação simples em Go (stdlib pura, sem dependências externas)
- Reconexão automática nativa do protocolo — zero código de retry no cliente
- Funciona sobre HTTP/2, que multiplexa conexões eficientemente
- Ações do oficial são REST puro (já definidas no Nível 4) — arquitetura mais simples e coesa
- Um único tipo de conexão persistente no sistema (vs. WebSocket que exigiria protocolo de upgrade e gestão de estado bidirecional)

**Negativas (trade-offs aceitos):**
- Unidirecional: ações do oficial usam endpoints REST separados (aceitável — já definido assim no Nível 4)
- HTTP/1.1 tem limite de 6 conexões simultâneas por domínio — mitigado com HTTP/2 (sem limite prático)
- Menos interativo que WebSocket para cenários de alta bidireci onalidade — não é o caso do CONTROL_OFFICER

## Alternativas Consideradas

**WebSocket:** Bidirecional, baixa latência. Descartado porque a bidirecionalidade não é necessária — as ações do oficial são REST e o feed é servidor→cliente. WebSocket adicionaria complexidade de protocolo de upgrade, gestão de estado de conexão e mensagens de heartbeat sem benefício real para este caso.
