# Nível 1 — Capacidades ✅ APROVADO

> **Status:** Aprovado pelo grupo  
> **Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra  
> **Data:** 08/06/2026

---

## Decisões Tomadas

### D1 — Escopo Geográfico do App
**Decisão:** O app BNP serve **apenas visitantes dentro do parque (geo-fenced)**.

- Funcionalidades dinâmicas (mapa ao vivo, avistamentos, botão de pânico) só ficam ativas quando o dispositivo está dentro do perímetro geográfico do BNP.
- Cadastro de conta pode ser realizado antes de entrar no parque.
- Fora do parque: app exibe apenas informações estáticas (fauna catalogada, regras, horários).
- **Implicação técnica:** o app precisará de lógica de geofencing no cliente para ativar/desativar funcionalidades.

---

### D2 — SLO de Avistamentos
**Decisão:** Avistamentos devem ser plotados no mapa e notificações disparadas em **≤ 30 segundos** após o registro.

- Alinhado com o enunciado original ("within a reasonable time after entry").
- Permite processamento assíncrono via event broker.
- O mesmo SLO se aplica à verificação de crowding: a decisão de suprimir ou publicar um avistamento deve ocorrer dentro dos mesmos 30 segundos.
- **Implicação técnica:** pipeline Sighting → Crowding Check → Notification deve completar em < 30s end-to-end.

---

### D3 — Botão de Pânico em Modo Degradado
**Decisão:** GPS + armazenamento local + reenvio automático quando reconectar.

- Ao pressionar o botão de emergência sem conectividade, o app:
  1. Captura as coordenadas GPS atuais e salva localmente.
  2. Exibe feedback visual: *"Alerta registrado. Aguardando conexão para envio..."*
  3. Monitora a conectividade e reenvia automaticamente assim que o sinal for restabelecido.
- **Implicação técnica:** o cliente mobile precisa de uma fila local persistente para eventos de emergência pendentes. O timestamp do evento deve ser o momento do acionamento, não do envio.

---

### D4 — Arquitetura de Clientes
**Decisão:** **Um único app mobile** com **roles diferentes** — UI adaptada conforme o perfil autenticado.

Roles previstos:
| Role | Acesso |
|------|--------|
| `VISITOR` | Mapa, avistamentos, GPS info, botão de pânico |
| `GUIDE` | Tudo do visitante + recebimento de alertas de emergência + localização de equipes |
| `MEDICAL_STAFF` | Recebimento de alertas de emergência + localização de incidente |
| `CONTROL_OFFICER` | Visão completa do parque: todos os carros, emergências ativas, despacho de equipes |

- **Implicação técnica:** o backend precisa emitir tokens com claims de role. A UI renderiza seções diferentes baseada no role. O role `CONTROL_OFFICER` terá uma tela de painel (dashboard) dentro do mesmo app.

---

### D5 — Autenticação
**Decisão:** Mesmo app, **perfis diferentes via login**.

- Um único sistema de autenticação (login/senha ou OAuth social para visitantes; credenciais internas para staff).
- O role é determinado no momento do login pelo backend e incluído no token de sessão.
- **Implicação técnica:** o User & Auth Service deve emitir tokens com role claim e o API Gateway deve validar permissões por endpoint.

---

### D6 — Sistemas Legados
**Decisão:** **Greenfield** — nenhuma integração com sistemas legados na v1.

- O BNP é desenvolvido do zero.
- Nenhuma dependência de sistemas externos obrigatória na v1.
- **Implicação de design:** a arquitetura deve prever pontos de extensão (event bus, webhooks) para futuras integrações, mas sem implementá-las agora.

---

## Capacidades IN SCOPE — v1

| # | Capacidade | Descrição |
|---|------------|-----------|
| C1 | Cadastro de conta | Visitantes e staff criam conta; visitantes vinculam telefone e placa do veículo |
| C2 | Informações geográficas | App exibe info do ecossistema/área baseada nas coordenadas GPS atuais |
| C3 | Registro de avistamento | Visitante registra animal + GPS + timestamp |
| C4 | Mapa de avistamentos | Avistamentos plotados no mapa em ≤ 30s para todos os usuários in-park |
| C5 | Notificação por tipo de animal | Visitante assina notificações de espécies específicas; recebe push quando avistadas |
| C6 | Controle de aglomeração | Sistema bloqueia plotagem/notificação quando threshold de carros na área é excedido |
| C7 | Botão de pânico | Dispara alerta de emergência com GPS; funciona em modo degradado (offline queue) |
| C8 | Despacho de emergência | CONTROL_OFFICER seleciona equipes por proximidade; guias/médicos recebem alerta |

## Capacidades FORA DO SCOPE — v1

- ❌ Acesso ao mapa ao vivo por usuários externos ao parque
- ❌ Integração com sistemas de rádio ou telecomunicação do parque
- ❌ Pagamento ou venda de ingressos via app
- ❌ Chat entre visitantes ou entre visitante e staff
- ❌ Rastreamento de animais (apenas avistamentos reportados por humanos)
- ❌ App web para visitantes (apenas mobile)
- ❌ Funcionamento completo 100% offline (apenas modo degradado para emergências)
