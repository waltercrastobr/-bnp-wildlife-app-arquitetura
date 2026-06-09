# Nível 2 — Componentes ✅ APROVADO

> **Status:** Aprovado pelo grupo  
> **Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra  
> **Data:** 08/06/2026  
> **Baseado em:** Decisões do Nível 1 (ver nivel_1_capacidades.md)

---

## Lista Definitiva de Componentes — v1

| # | Componente | Responsabilidade única |
|---|------------|------------------------|
| 1 | **Mobile App** | Interface única para todos os roles (VISITOR, GUIDE, MEDICAL_STAFF, CONTROL_OFFICER). Renderiza UI adaptada por role. Captura GPS, registra avistamentos, aciona pânico, exibe mapa e notificações. |
| 2 | **API Gateway** | Ponto de entrada único do sistema. Roteia requisições para os serviços internos, valida tokens de autenticação e aplica rate limiting. |
| 3 | **Auth & User Service** | Gerencia cadastro de contas, autenticação, emissão de tokens com claims de role, e vínculo de telefone/placa do veículo ao perfil do visitante. |
| 4 | **Park Information Service** | Fornece dados do ecossistema e geografia do parque com base em coordenadas GPS. Serve informações estáticas e semi-estáticas (áreas, flora, fauna catalogada). |
| 5 | **Location Tracker Service** | Recebe e armazena as atualizações contínuas de posição GPS de todos os veículos e membros de staff ativos no parque. Mantém o estado de "quem está onde agora". |
| 6 | **Sighting Service** | Recebe eventos de avistamento (animal + GPS + timestamp) submetidos por visitantes. Valida e persiste o avistamento. Não decide sobre publicação — delega isso ao fluxo de crowding. |
| 7 | **Crowding Prevention Engine** | Consome avistamentos validados e consulta o Location Tracker para contar veículos no raio do avistamento. Decide se o avistamento deve ser publicado ou suprimido com base no threshold configurado. |
| 8 | **Map Service** | Serve as tiles cartográficas do parque (mapas base, trilhas, áreas de ecossistema). Permite download antecipado para uso semi-offline dentro do parque. |
| 9 | **Notification Service** | Gerencia assinaturas de visitantes por tipo de animal. Despacha notificações para inscritos quando um avistamento é aprovado. Possui fila interna de prioridade: `CRITICAL` (emergências) e `STANDARD` (avistamentos). |
| 10 | **Emergency & Control Service** | Gerencia o ciclo completo de emergências: recebe alertas de pânico, consulta o Location Tracker para identificar guias/médicos mais próximos, notifica as equipes e disponibiliza interface de coordenação para o CONTROL_OFFICER. Roda com recursos computacionais dedicados e isolados. |
| 11 | **Data Layer** | Camada de persistência. Detalhada no Nível 5 (tecnologias). Inclui: armazenamento de usuários/veículos, histórico de avistamentos, estado de localização em tempo real, tiles de mapa e logs de emergência. |

---

## Diagrama C4 — Nível Container (textual)

```
┌─────────────────────────────────────────────────────────────────┐
│                    BNP Wildlife App — Sistema                    │
│                                                                 │
│  ┌──────────────────┐                                           │
│  │   Mobile App     │ ← role-based UI (VISITOR/GUIDE/          │
│  │  (iOS/Android)   │   MEDICAL_STAFF/CONTROL_OFFICER)         │
│  └────────┬─────────┘                                           │
│           │                                                     │
│  ┌────────▼─────────┐                                           │
│  │   API Gateway    │ ← autenticação, roteamento, rate limit   │
│  └────────┬─────────┘                                           │
│           │                                                     │
│  ┌────────┴──────────────────────────────────────┐             │
│  │                 Microserviços                  │             │
│  │                                               │             │
│  │  ┌─────────────────┐  ┌─────────────────────┐ │             │
│  │  │Auth & User Svc  │  │Park Information Svc │ │             │
│  │  └─────────────────┘  └─────────────────────┘ │             │
│  │                                               │             │
│  │  ┌─────────────────┐  ┌─────────────────────┐ │             │
│  │  │Location Tracker │  │   Sighting Service  │ │             │
│  │  └─────────────────┘  └─────────────────────┘ │             │
│  │                                               │             │
│  │  ┌─────────────────┐  ┌─────────────────────┐ │             │
│  │  │Crowding Prevent.│  │  Notification Svc   │ │             │
│  │  │    Engine       │  │  [CRITICAL / STD]   │ │             │
│  │  └─────────────────┘  └─────────────────────┘ │             │
│  │                                               │             │
│  │  ┌─────────────────┐  ┌─────────────────────┐ │             │
│  │  │   Map Service   │  │Emergency & Control  │ │             │
│  │  │  (map tiles)    │  │Svc ⚠️ [ISOLADO]     │ │             │
│  │  └─────────────────┘  └─────────────────────┘ │             │
│  └───────────────────────────────────────────────┘             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     Data Layer                          │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Decisões e Rationale

### D2.1 — Crowding Prevention Engine independente (Opção B)
Separado do Sighting Service para garantir responsabilidade única. A lógica de crowding (threshold por área, configurabilidade futura) pode evoluir sem impactar o registro de avistamentos. Permite escalar o engine de crowding de forma independente em períodos de alta visitação.

### D2.2 — Location Tracker Service dedicado (Opção B)
Sem um rastreador contínuo, o sistema de crowding seria impreciso (só contaria carros que registraram avistamentos, não todos os presentes). O Location Tracker também é essencial para o Emergency & Control Service identificar guias e médicos mais próximos em tempo real. É o "estado vivo do parque".

### D2.3 — Map Service próprio no backend (Opção A)
Garante controle total sobre os mapas do parque (trilhas internas, áreas de ecossistema, zonas restritas). Permite que o app faça download antecipado dos tiles ao entrar no parque, viabilizando uso semi-offline — consistente com a decisão D3 do Nível 1 (modo degradado).

### D2.4 — Notification Service único com filas de prioridade HIGH/LOW (Opção C)
Solução de equilíbrio: mantém um único serviço de notificação (menor complexidade operacional) mas garante que alertas de emergência nunca aguardem na fila de avistamentos. A fila `CRITICAL` tem processamento prioritário e sem throttle.

### D2.5 — Emergency & Control Service isolado com recursos dedicados (Opção A)
Critério definido no Nível 1: o botão de emergência JAMAIS pode falhar por sobrecarga de outros serviços. Isolamento físico (container/processo separado com CPU e memória garantidos) é a única forma de assegurar isso. Mesmo que o Sighting Service e o Crowding Engine estejam sobrecarregados, o Emergency Service continua operando.
