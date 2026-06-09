# ADR-004: Apps Nativos (Swift + Kotlin) para o Mobile

**Status:** Aceito  
**Data:** 08/06/2026  
**Grupo:** Walter Monteiro · Claudino Neto · Vinícius Seabra

---

## Contexto

O app BNP mobile tem requisitos que testam os limites das plataformas móveis:
1. **GPS em background contínuo** — envio de localização a cada 30s mesmo com app em background (D3.2)
2. **Geofencing** — detectar entrada/saída do perímetro do parque para ativar/desativar funcionalidades (D1.1)
3. **Fila offline de emergência** — persistir alerta de pânico localmente e reenviar ao reconectar (D1.3)
4. **Push notifications críticas** — alertas de emergência para GUIDE e MEDICAL_STAFF nunca podem ser ignorados pelo sistema operacional
5. **SSE persistente** — manter conexão aberta com o servidor para o feed do CONTROL_OFFICER

Esses requisitos demandam acesso profundo às APIs nativas de cada plataforma — especialmente o GPS em background, que iOS e Android tratam de formas radicalmente diferentes e restritivas.

## Decisão

Adotamos **apps nativos separados**: Swift para iOS e Kotlin para Android.

- **iOS:** CoreLocation Manager com `allowsBackgroundLocationUpdates = true` e a opção `UIBackgroundModes: location` no Info.plist. Geofencing via `CLCircularRegion`. Persistência offline via Core Data. Push via APNs.
- **Android:** `FusedLocationProviderClient` com `PRIORITY_HIGH_ACCURACY` em um Foreground Service. Geofencing via `GeofencingClient`. Persistência offline via Room Database (SQLite). Push via FCM.

Os dois apps implementam **os mesmos contratos do Nível 4** — qualquer divergência de comportamento entre iOS e Android é tratada como bug.

## Consequências

**Positivas:**
- Acesso irrestrito a todas as APIs nativas de GPS, geofencing e notificações push
- Background GPS confiável — frameworks cross-platform frequentemente têm limitações ou workarounds específicos por plataforma
- Performance máxima — sem camada intermediária de abstração
- Notificações críticas de emergência com categorias de prioridade máxima (iOS Critical Alerts, Android HIGH_PRIORITY)

**Negativas (trade-offs aceitos):**
- Dois codebases: dobra o esforço de implementação e manutenção
- Qualquer mudança de contrato (Nível 4) precisa ser implementada em ambos os apps
- Custo maior de desenvolvimento — justificado pelos requisitos críticos de segurança do botão de pânico

## Alternativas Consideradas

**React Native:** Cross-platform com JavaScript. Descartado porque o suporte a background GPS no iOS via React Native requer bibliotecas de terceiros (`react-native-background-geolocation`) com comportamentos não-determinísticos em versões novas do iOS. Para um sistema de emergência, confiabilidade é inegociável.

**Flutter:** Cross-platform com Dart. Performance excelente, mas o mesmo problema de background GPS — plugins de geolocalização (`geolocator`, `background_fetch`) têm histórico de quebrar a cada atualização de iOS. O botão de pânico não pode depender de um plugin de terceiro.
