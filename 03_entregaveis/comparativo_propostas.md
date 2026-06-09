# Comparativo: Proposta Anterior vs. Nova Proposta (Design-First)

> **Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra  
> **Disciplina:** Arquitetura de Sistemas — Prof. Kiev Gama  
> **Data:** 08/06/2026  
>
> ⚠️ **Este documento é o esqueleto de referência.**  
> As opiniões e análise crítica devem ser escritas pelo grupo, com suas próprias palavras — sem uso de IA.

---

## Tabela Comparativa Geral

| Aspecto | Proposta Anterior (Gemini rápido) | Nova Proposta (Design-First) |
|---------|-----------------------------------|------------------------------|
| **Processo** | IA gerou tudo de uma vez a partir de um único prompt | 5 níveis progressivos com aprovação explícita do grupo a cada etapa |
| **Participação do grupo** | Passiva — recebemos o resultado e avaliamos depois | Ativa — cada decisão de design foi escolhida pelo grupo com opções apresentadas |
| **Tempo até o primeiro diagrama** | Imediato | Apenas no Nível 5 — após aprovação de capacidades, componentes, interações e contratos |
| **Componentes definidos** | 6 microserviços (sem Location Tracker, sem Map Service) | 10 componentes com responsabilidades precisas |
| **Location Tracker** | Ausente — Redis "mágico" sem definir quem escreve nele | Componente dedicado com contrato explícito de lote de GPS |
| **Map Service** | Não mencionado — assumia-se provedor externo | Serviço próprio que serve tiles com suporte a download offline |
| **Crowding Prevention** | Descrito como componente, mas sem decisão sobre isolamento | Serviço independente com responsabilidade única, escalável separadamente |
| **Emergency Service** | Descrito textualmente, sem SLA nem fluxo detalhado | Isolado em namespace Kubernetes com recursos dedicados; SLA definido no Nível 3 |
| **Contratos de API** | Não existiam | 9 endpoints definidos com schemas de request/response |
| **Eventos do barramento** | "Kafka/RabbitMQ" mencionado sem definir tópicos ou schemas | 5 eventos definidos com schema JSON, fila e produtor/consumidor |
| **ADRs** | Ausentes | 5 ADRs com contexto, decisão, consequências e alternativas rejeitadas |
| **Diagrama** | Mindmap gerado no XMind | C4 Model em 3 níveis (Contexto + Container + Component) |
| **Mobile** | "Aplicativo móvel" genérico | Swift (iOS) + Kotlin (Android), nativos, com justificativa técnica |
| **Backend** | Não especificado | Go (Golang) com justificativa por perfil de carga de cada serviço |
| **Banco de dados** | PostgreSQL+PostGIS, MongoDB, Redis — listados sem definir qual serviço usa qual | PostgreSQL+PostGIS para relacional/geoespacial, MongoDB para documentos — mapeamento explícito por serviço |
| **Token/Autenticação** | OAuth2/JWT mencionado | Schema do token definido com campos e regras de expiração |
| **Modo offline** | Não tratado | Fila local de emergência com deduplicação por emergencyId definida no Nível 1 |
| **SLO de avistamentos** | "Reasonable time" sem número | ≤ 30 segundos, definido no Nível 1 e rastreado no pipeline do Nível 3 |

---

## Diferenças Arquiteturais Detalhadas

### 1. Descoberta do Location Tracker — o "estado vivo do parque"

**Proposta anterior:** Redis armazenava posições de GPS, mas nenhum componente era responsável por escrever nele. A proposta assumia que de alguma forma os dados chegavam lá.

**Nova proposta:** O Location Tracker Service é um componente dedicado com contrato explícito: o app envia lotes de posição a cada 30s (`POST /locations/batch`), o serviço processa e persiste em PostgreSQL+PostGIS. O Crowding Engine e o Emergency Service consultam o Location Tracker via chamadas síncronas. Toda a cadeia de responsabilidade está mapeada.

**Impacto:** Sem o Location Tracker explícito, o sistema de crowding da proposta anterior seria funcionalmente impossível de implementar — ninguém sabia de onde viriam os dados de localização dos veículos.

---

### 2. Fluxo de Emergência com SLA

**Proposta anterior:** "O serviço identifica os alojamentos médicos e guias mais próximos via Redis, notifica-os imediatamente" — texto descritivo sem definição de como, com que latência, ou o que acontece sem sinal.

**Nova proposta:** Fluxo completo com diagrama de sequência, tratamento de modo offline (fila local + reenvio), deduplicação por `emergencyId`, isolamento físico do serviço em namespace Kubernetes com recursos garantidos, e SLA implícito de < tempo do ciclo de processamento da fila CRITICAL.

---

### 3. Decisão de Tecnologia com Justificativa vs. Lista de Buzzwords

**Proposta anterior:** "Apache Kafka / RabbitMQ" — apresentados como alternativas equivalentes sem critério de escolha.

**Nova proposta:** Kafka escolhido sobre RabbitMQ com justificativa explícita (retenção de eventos de emergência para auditoria), trade-offs documentados (mais pesado para operar) e reconhecimento de que é overkill para o volume atual — decisão consciente, não ingênua.

---

### 4. Geofencing como Capacidade Fundamental

**Proposta anterior:** Não mencionado.

**Nova proposta:** Definido no Nível 1 como requisito fundamental (D1.1): funcionalidades dinâmicas só ativam quando o dispositivo está dentro do perímetro. Isso impacta diretamente o app nativo (CoreLocation/GeofencingAPI), a estratégia de download offline de tiles de mapa, e a lógica de ativação do Location Tracker em background.

---

### 5. App Único vs. Separação de Clientes

**Proposta anterior:** Implicava separação ("aplicativo móvel" para visitantes, painel web para Centro de Controle).

**Nova proposta:** Decisão explícita e justificada no Nível 2: um único app com UI baseada em role — reduz o esforço de desenvolvimento. O CONTROL_OFFICER usa o mesmo app com tela de painel, alimentada pelo canal SSE dedicado.

---

## Avaliação de Qualidade — Comparativo

| Critério | Proposta Anterior | Nova Proposta | Evolução |
|----------|-------------------|---------------|----------|
| Disponibilidade do Emergency | Alta — serviço isolado mencionado | Crítica — isolamento implementado em Kubernetes com recursos dedicados | ✅ Concretizado |
| Desempenho/Latência | "Redis garante ms" sem definir quem popula o Redis | Pipeline definido end-to-end com SLO de 30s rastreável | ✅ Mensurável |
| Escalabilidade | Mencionada como vantagem genérica de microserviços | Cada serviço com réplicas definidas na visão de deploy | ✅ Operacionalizável |
| Consistência de Dados | Eventual para mapa, imediata para cadastro | Mantido + explicitado por banco (PostgreSQL ACID para usuários, MongoDB eventual para avistamentos) | ✅ Mais preciso |
| Segurança | OAuth2/JWT mencionado | Schema de token com campos, expiração e lista negra definidos | ✅ Implementável |
| Manutenibilidade | Implícita pela arquitetura de microserviços | ADRs documentam o "porquê" de cada decisão — próximos devs entendem o raciocínio | ✅ Documentada |
| Modo Offline | Ausente | Fila local de emergência com deduplicação e feedback visual | ✅ Novo requisito coberto |

---

## Espaço para Opiniões do Grupo

> *Esta seção deve ser preenchida pelos integrantes do grupo com suas próprias palavras. O professor Kiev Gama pediu explicitamente a opinião de vocês — não do LLM.*

### Walter Monteiro
*(escreva sua opinião aqui)*

### Claudino Neto
*(escreva sua opinião aqui)*

### Vinícius Seabra
*(escreva sua opinião aqui)*

---

### Perguntas norteadoras para as opiniões
1. A abordagem Design-First (5 níveis) mudou a qualidade das decisões tomadas? Por quê?
2. Quais decisões foram as mais difíceis de tomar? O que o processo de escolher entre opções concretas trouxe de diferente?
3. A proposta anterior foi gerada pela IA sem nenhuma dessas perguntas — qual o risco disso em um projeto real?
4. O Location Tracker "invisível" na proposta anterior é um exemplo da "Implementation Trap" do blog? Como vocês enxergam isso?
5. A separação em 5 níveis progressivos custou mais tempo. Valeu a pena? Em que contextos você usaria isso?
6. Houve alguma decisão que vocês tomariam diferente agora, olhando para o resultado final?
