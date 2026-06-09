# ADR-002: Estratégia de Banco de Dados — PostgreSQL+PostGIS + MongoDB

**Status:** Aceito  
**Data:** 08/06/2026  
**Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra

---

## Contexto

O sistema BNP possui duas categorias distintas de dados com requisitos muito diferentes:

**Dados relacionais e geoespaciais:**
- Usuários, veículos, autenticação → relacionamentos fortes, ACID necessário
- Posições ao vivo de veículos e staff → queries geoespaciais frequentes e de baixa latência (`ST_DWithin`, `ST_Distance`)
- Áreas do parque (ecossistemas, zonas restritas) → polígonos geográficos

**Dados semi-estruturados:**
- Histórico de avistamentos → estrutura variável (notas opcionais, espécies diversas com metadados diferentes)
- Catálogo de espécies → documentos ricos com fotos, descrições, comportamentos
- Logs de emergência → timeline de eventos encadeados de estrutura variável

A grande pergunta era: usar um banco único para tudo, ou separar por tipo de dado?

## Decisão

Adotamos **estratégia polyglot**:
- **PostgreSQL 16 + PostGIS 3** para dados relacionais e geoespaciais (usuários, veículos, localização ao vivo, áreas do parque)
- **MongoDB 7** para dados de documentos (avistamentos, catálogo de espécies, logs de emergência)

O Location Tracker usa o PostgreSQL+PostGIS (não Redis) para manter as posições ao vivo. A decisão foi baseada no fato de que o PostgreSQL com PostGIS já está no sistema — adicionar Redis como terceiro banco seria complexidade desnecessária para o volume de dados esperado.

## Consequências

**Positivas:**
- PostGIS oferece as queries geoespaciais mais maduras e expressivas do mercado: `ST_DWithin`, `ST_Within`, `<->` para ordenação por distância — exatamente o que o Crowding Engine e o Emergency Service precisam
- MongoDB permite evolução do schema de avistamentos sem migrations — adicionar novos campos de espécies não requer alterar tabelas
- Separação de responsabilidades: dados críticos de identidade no banco ACID (PostgreSQL), dados de volume e flexibilidade no banco de documentos (MongoDB)

**Negativas (trade-offs aceitos):**
- Dois sistemas de banco para operar e monitorar
- Queries que cruzam os dois bancos (ex: avistamento + dados do visitante) precisam ser feitas na camada de aplicação, não via JOIN — maior responsabilidade dos serviços
- PostgreSQL para posições ao vivo tem latência maior que Redis (~1-5ms vs <1ms), aceitável para o SLO de 30s dos avistamentos e para o fluxo de emergência (que consulta localização uma única vez, de forma síncrona)

## Alternativas Consideradas

**PostgreSQL+PostGIS único (sem MongoDB):** Eliminaria a complexidade de dois bancos. Descartado porque schemas rígidos para avistamentos limitariam a evolução do catálogo de espécies e a estrutura de timeline dos logs de emergência.

**Redis para Location Tracker:** Latência sub-milissegundo e comandos `GEORADIUS` nativos. Descartado para evitar um terceiro sistema de banco de dados, dado que o volume de veículos simultâneos (dezenas a centenas) não justifica a complexidade adicional de Redis. PostGIS com índice geoespacial e conexão poolada atende o SLO.
