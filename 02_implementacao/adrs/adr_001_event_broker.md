# ADR-001: Apache Kafka como Barramento de Eventos

**Status:** Aceito  
**Data:** 08/06/2026  
**Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra

---

## Contexto

O sistema BNP possui um pipeline de processamento assíncrono definido no Nível 3:  
`Sighting Service → Crowding Prevention Engine → Notification Service`

Além disso, alertas de emergência precisam de uma fila com prioridade máxima (`CRITICAL`) que garanta processamento imediato, independentemente do volume de eventos de avistamento em curso.

Precisávamos de um barramento de eventos que suportasse:
1. Múltiplos tópicos com consumidores independentes
2. Diferenciação de prioridade entre filas (CRITICAL vs. STANDARD)
3. Retenção de mensagens para reprocessamento em caso de falha de consumidor
4. Capacidade de escalar para dezenas a centenas de mensagens por segundo durante picos de visitação

## Decisão

Adotamos **Apache Kafka 3.x com KRaft** (modo sem Zookeeper) como barramento de eventos distribuído.

A diferenciação CRITICAL/STANDARD é implementada via **tópicos separados** com grupos de consumidores distintos e configuração de `max.poll.records=1` no consumidor de emergências, garantindo processamento imediato de cada alerta sem agrupamento em lote.

## Consequências

**Positivas:**
- Retenção de mensagens por 7-30 dias permite reprocessamento e auditoria completa de emergências
- Escala horizontal via partições — picos de avistamento não degradam o pipeline de emergência
- Desacoplamento total entre Sighting Service e Crowding Engine: uma falha em um não impacta o outro
- KRaft elimina a dependência do Zookeeper, simplificando a operação

**Negativas (trade-offs aceitos):**
- Kafka é mais complexo de operar que um message broker simples
- Latência de entrega de mensagens é maior que RabbitMQ para volumes baixos (~5-15ms vs ~1-3ms)
- Para o volume atual do BNP (dezenas de eventos/segundo), Kafka é tecnicamente overkill — a escolha é justificada pela escalabilidade futura e pela retenção de eventos de emergência para auditoria

## Alternativas Consideradas

**RabbitMQ:** Suporte nativo a filas de prioridade (`x-max-priority`), mais simples de operar, menor latência para volumes baixos. Descartado principalmente pela ausência de retenção de mensagens após consumo — inaceitável para logs de emergência que precisam de auditoria persistente.
