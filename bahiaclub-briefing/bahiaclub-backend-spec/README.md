---
name: project-agentes-bahia
description: "Sistema completo de agentes IA y funciones para bahiaclub.mx — 15 agentes, plataforma de reservas, funciones adicionales, stack, parámetros y orden de implementación"
metadata: 
  node_type: memory
  type: project
  originSessionId: 885b2381-9afd-4dd1-8f46-70470ea8a90d
---

# Sistema de Agentes IA — bahiaclub.mx

## Parámetros del sistema
- **Miembros**: 200–500 activos
- **Operador**: 1 admin/gerente dedicado — flujos deben ser "aprobar/rechazar", mínima intervención manual
- **Canchas**: 8 pádel · 3 tenis arcilla (abiertas) · 3 tenis dura (abiertas) · 8 pickleball = 22 total
- **Estado del club**: ya abierto y operando
- **Software actual**: ninguno — todo manual en Excel/WhatsApp. Los agentes son el primer sistema formal.
- **Restaurante**: concesionado — excluido del sistema completamente
- **Tienda/pro shop**: concesionada — excluida

## Stack técnico base (compartido por todos los agentes)

| Capa | Tecnología |
|---|---|
| Database | Supabase (PostgreSQL + real-time) |
| Backend API | Vercel serverless functions (`/api/*`) |
| AI | Claude API — claude-sonnet-4-6 |
| WhatsApp | Twilio WhatsApp API |
| IG / FB DMs | Meta Messenger API (requiere Meta Business — pendiente confirmar) |
| Google Calendar | Google Calendar API + Service Account |
| Open Banking (futuro) | Belvo (México) |
| Trending data | Apify + Google Trends API |
| Auth socios/admin | Magic link por email (Supabase Auth) |
| Auth miembros app | OTP por WhatsApp |
| Clima | Tomorrow.io API |

## Pendiente de confirmar
- **Meta Business Account**: Agentes 2, 5 y 12 comparten la misma Meta Business App. Gonzalo necesita verificar si Bahía Club ya tiene cuenta activa.
- **Control de acceso G**: Días de gracia y umbral de bloqueo los define gerencia antes de activar.

---

## Los 15 Agentes

### Agente 1 — Check-in Tracker (Seguimiento de Membresías)
**Qué hace**: Registra cada entrada al club via lector QR/NFC. Detecta inactivos (30/60/90 días sin visita). Genera alertas. Dashboard de asistencia.
**Flujo**: Miembro escanea QR/NFC → tablet en recepción → `POST /api/checkin` → Supabase → cron diario analiza patrones → alertas en dashboard.
**Tablas**: `members`, `check_ins`, `alerts`
**QR Scanner**: `bahiaclub.mx/scanner` — usa `html5-qrcode`, modo kiosko, feedback visual inmediato.
**Oleada**: 1 (primero — genera el dato fundamental de quién viene y cuándo)

---

### Agente 2 — Marketing Intelligence
**Qué hace**: Monitorea tendencias en redes sociales en tiempo real, filtradas por Bahía de Banderas/Riviera Nayarit. Sugiere contenido a crear. Se conecta a Meta Ads para difundir contenido al mercado objetivo.
**Flujo**: Cron continuo → APIs de tendencias (TikTok, IG, X) → Claude filtra por relevancia → genera sugerencias (copy, formato, hashtags, presupuesto) → admin aprueba → Meta Ads API lanza campaña.
**Integraciones**: Meta Business API, Apify (scraper trends), Google Trends API.
**Tablas**: `trending_signals`, `content_suggestions`, `campaign_performance`
**Oleada**: 4 (requiere Meta Business confirmado)

---

### Agente 3 — Cobranza (Billing & Collections)
**Qué hace**: Base de datos central de pagos (inscripciones y mensualidades). El admin registra pagos manualmente (tarjeta/efectivo/transferencia). Genera presupuesto real vs. esperado por mes. Recordatorios automáticos vía WhatsApp para vencidos.
**Flujo**: Admin registra pago → Supabase → cron diario detecta vencidos → Claude genera mensaje → Twilio WhatsApp → registra en historial.
**Lógica de recordatorios**:
- 1–5 días atraso: recordatorio amable
- 6–15 días: seguimiento directo con opción de pago
- +15 días: alerta al admin + mensaje urgente
- +30 días: flag crítico — posible baja
**Tablas**: `payments`, `billing_messages`, `monthly_budget`
**WhatsApp**: Twilio (para producción escalar a Meta WhatsApp Business API directo)
**Oleada**: 1

---

### Agente 4 — Planeación de Eventos (Event Intelligence)
**Qué hace**: Analiza patrones de tráfico del Agente 1 para detectar días/horas de alta y baja demanda. Responde consultas de planeación ("¿cuándo sería mejor un torneo?"). Propone eventos proactivamente cuando detecta caídas de flujo. Loop de aprendizaje: el equipo registra resultados de cada evento y el agente mejora sus sugerencias con el tiempo (RAG).
**Flujo**: Cron semanal → analiza check_ins → Claude cruza con historial de eventos → genera propuestas → admin aprueba → evento agendado → post-evento: admin registra outcome → se guarda para próxima sugerencia.
**RAG**: Cada `event_outcome` se convierte en contexto para Claude al generar la siguiente propuesta.
**Tablas**: `events`, `event_outcomes`, `traffic_patterns`
**Dashboard**: Heatmap días×horas, propuestas del agente, historial con ratings, chat de consulta.
**Oleada**: 3

---

### Agente 5 — Prospección (Sales Concierge)
**Qué hace**: Atiende prospectos 24/7 en WhatsApp e Instagram/Facebook DMs. Califica el lead. Resuelve dudas sobre planes, precios e instalaciones. Agenda citas en Google Calendar. Genera propuesta personalizada. Escala a humano cuando el prospecto está listo para cerrar.
**Canales**: WhatsApp (Twilio) + IG/FB DMs (Meta Messenger API — misma app que Agentes 2 y 12).
**Knowledge base**: Planes y precios, instalaciones, ubicación, horarios, FAQ, tono de marca.
**Escalada**: Si no sabe algo → "Lo verifico y te confirmo" → alerta al admin.
**Google Calendar**: `calendar.app.google/cedvSmtcwGR3grVc6` (ya existe en el sitio).
**Tablas**: `prospects`, `prospect_messages`, `prospect_appointments`
**Oleada**: 2

---

### Agente 6 — Concierge (Reservaciones)
**Qué hace**: Gestiona reservas de canchas y otras instalaciones. Canal principal: WhatsApp (lenguaje natural). Canal secundario: portal web `bahiaclub.mx/reservar`. Verifica membresía activa y pagos antes de confirmar. Confirmación + recordatorio 1h antes.
**Lenguaje natural**: Entiende "reserva pádel mañana a las 7am para 4", "cancela mi reserva de mañana", "¿tengo cancha hoy?"
**Validaciones antes de confirmar**: disponibilidad, membresía activa, pagos al corriente (Agente 3), sin duplicados.
**Canchas disponibles**: 8 pádel, 3 tenis arcilla, 3 tenis dura, 8 pickleball + restaurante + área social.
**Portal web**: OTP por WhatsApp, vista semana con slots coloreados, un clic reserva.
**Tablas**: `facilities`, `bookings`, `blackout_slots`
**Oleada**: 1

---

### Agente 7 — Contabilidad
**Qué hace**: Consolida ingresos (del Agente 3 + fuentes fijas + upload Excel) y egresos (manuales o banco) en estado financiero mes a mes. Dashboard en vivo con P&L, flujo de caja y presupuesto vs. real. Acceso diferenciado por rol.
**Fuentes de ingreso**: Agente 3 (automático), ingresos fijos configurados, upload Excel, gastos bancarios (Belvo, futuro).
**Roles de acceso**:
- Gerencia/Consejo: P&L completo, cada ingreso y egreso, exportar reportes
- Socios-dueños: solo resumen ejecutivo (3 números grandes + gráfica + narrative Claude)
- Admin operativo: captura gastos/ingresos, no ve reporte de socios
**Auth**: Magic link por email por socio.
**Reportes**: P&L mensual, flujo de caja, presupuesto vs. real, proyección del mes en curso.
**Tablas**: `income_entries`, `expense_entries`, `monthly_summary`, `stakeholders`
**Oleada**: 3

---

### Agente 8 — CFO / Inteligencia Financiera
**Qué hace**: Proyecciones y escenarios ("¿qué pasa si subo precios 10%?"), alertas tempranas de caída de ingresos antes de que ocurran, optimización de precios basada en ocupación, reporte para inversionistas/deck financiero. Monitorea tendencias de mercado en tiempo real y estacionalidad de Riviera Nayarit.
**Estacionalidad**: Temporada alta dic–abr, baja may–oct. Integrado en todas las proyecciones.
**Datos externos**: Google Trends API, DATATUR (turismo RN), Banxico (tipo de cambio), benchmarks del sector.
**Chat con CFO**: Socios del consejo pueden preguntar en lenguaje natural: "¿cuánto necesito para abrir otra sede?".
**Tablas**: `projections`, `scenarios`, `market_signals`, `alerts_financial`
**Consume datos de**: Todos los agentes (1, 3, 4, 5, 6, 7)
**Oleada**: 4

---

### Agente 9 — Satisfacción de Miembros (NPS Intelligence)
**Qué hace**: Envía UNA encuesta mensual por miembro — no después de cada visita. Detecta el momento óptimo: cuando el miembro tuvo check-in reciente pero no ha recibido encuesta este mes. Analiza respuestas con Claude. Escala insatisfechos al equipo.
**Lógica de timing**: Cron diario → ¿check-in últimos 3 días? → ¿ya recibió encuesta este mes? → si cumple: envía.
**Encuesta**: 1 pregunta NPS (1-10) + pregunta abierta adaptada al score.
**Señales de churn**: NPS < 6 → alerta inmediata; NPS bajó 3+ puntos → alerta amarilla; sin respuesta 2 meses → desenganchado.
**Regla**: Si el miembro tiene pago atrasado ese mes → no recibe encuesta.
**Tablas**: `nps_surveys`, `nps_alerts`, `nps_summary`
**Oleada**: 2

---

### Agente 10 — Mantenimiento e Inventario
**Qué hace**: Tres fuentes de input: reportes de miembros (WhatsApp/portal), reportes manuales del staff, y calendario de mantenimiento preventivo programado. Prioriza tickets con Claude. Gestiona proveedores. Inventario de equipos y consumibles.
**Prioridades**: 🔴 Urgente (seguridad/no operable) → notifica gerencia inmediato | 🟡 Alta (afecta experiencia) → 24-48h | 🟢 Normal (preventivo/estético) → en la semana.
**Integración clave**: Ticket urgente en cancha → bloquea automáticamente en `blackout_slots` del Agente 6.
**Preventivo configurable**: Admin define frecuencia por instalación (ej. pádel mensual, alberca semanal). El agente genera tickets automáticamente.
**Tablas**: `maintenance_tickets`, `maintenance_schedule`, `inventory_items`, `inventory_alerts`
**Oleada**: 2

---

### Agente 11 — Staff y Horarios
**Qué hace**: Gestiona turnos del personal. Cruza con picos de demanda del Agente 1. Genera plantilla semanal como propuesta (admin aprueba). Staff reporta disponibilidad/ausencias por WhatsApp. Detecta sobrecargas y huecos de cobertura.
**Flujo**: Cron lunes → toma demanda proyectada + disponibilidad del staff → genera plantilla → alerta conflictos → admin aprueba → notifica a cada empleado por WA.
**Tablas**: `staff`, `shifts`, `availability`, `coverage_alerts`
**Oleada**: 3

---

### Agente 12 — Reputación Digital
**Qué hace**: Monitorea Google Reviews, TripAdvisor y menciones en IG/FB. Claude genera respuesta empática a reseñas negativas para que el admin apruebe con un clic. Responde positivas automáticamente (con variaciones). Alerta cuando score baja o mención negativa tiene alto alcance.
**Canales**: Google My Business API, Apify (TripAdvisor), Meta Graph API (misma app que Agentes 2 y 5).
**Tablas**: `reviews`, `reputation_metrics`, `reputation_alerts`
**Oleada**: 4 (requiere Meta Business confirmado)

---

### Agente 15 — Clases e Instrucción
**Qué hace**: Catálogo de clases (pádel, tenis, pickleball, gym). Miembro reserva clase por WhatsApp o portal. Gestiona disponibilidad de instructores (conectado al Agente 11). Trackea progresión del alumno. Claude genera resumen post-clase.
**Tipos**: Individual, grupal, clínica, torneo interno.
**Flujo**: Miembro pide clase → consulta disponibilidad de instructores + canchas (Agente 6) → propone opciones → confirma → reserva + bloquea cancha + notifica instructor → post-clase: instructor registra notas → Claude genera resumen de progresión.
**Tablas**: `class_catalog`, `class_bookings`, `member_progression`
**Oleada**: 2

---

## Plataforma de Reservas — reemplaza Playtomic (ahorra $50k MXN/mes)

**URL**: `app.bahiaclub.mx` (subdominio separado del sitio de marketing)
**Stack**: Next.js + mismos endpoints Vercel + Supabase Realtime
**Auth**: OTP por WhatsApp

### Funciones
1. **Reserva de canchas** — 22 canchas, filtro por deporte, disponibilidad en tiempo real, conectado al Agente 6
2. **Buscar pareja / completar grupo** — reserva "abierta" visible para otros miembros, notificación WA cuando alguien se une
3. **Torneos** — inscripción, bracket automático, resultados, ranking en tiempo real, conectado al Agente 4
4. **Perfil del jugador** — foto, nivel por deporte, historial, estadísticas
5. **Ligas permanentes (Función K)** — integradas en la misma app
6. **Logros y badges (Función L)** — visibles en el perfil

**Sin pasarela de pago** — reservas incluidas en membresía.

**Futuro app nativa**: Si el club lo decide, `app.bahiaclub.mx` se convierte en iOS/Android con acceso a TODO el sistema (cobranza, clases, NPS, etc.). El backend ya existe.

---

## Funciones Adicionales

### Función A — Programa de Lealtad (Puntos)
Miembros acumulan puntos por: check-in (10pts), clase asistida (25pts), torneo (50pts), ganar torneo (30pts extra), responder NPS (15pts), cumpleaños (100pts), referir miembro (500pts), 6 meses activos (200pts).
Canjean por: clases gratis, descuento en inscripción de invitado, merch del club, mes con descuento, acceso a evento exclusivo.
Valores configurables por el admin.

### Función B — Day Pass ($500 MXN/día — embudo a membresía)
Visitante paga $500 → se registra (nombre + teléfono mínimo) → accede por el día → al final del día recibe encuesta de satisfacción vía WA → si score ≥ 7 → Claude manda propuesta personalizada de membresía calculando breakeven ("con 13 visitas/mes ya pagas lo mismo que Individual") → lead pasa al Agente 5.
Ingresos: registrados en Agente 3 y Agente 7 separado de membresías.
KPI en Agente 8: tasa de conversión day pass → membresía.

### Función C — Comunicaciones Segmentadas
Mensajes internos del club a miembros (≠ Agente 2 que es marketing externo). Tipos: aviso operativo, novedad del club, campaña de reactivación, recordatorio de evento, comunicado a socios.
Segmentación: por plan, deporte favorito, frecuencia de visita, inscripción a evento, todos.
Canal: WhatsApp o push notification en app.
Integración: ticket urgente Agente 10 → notifica miembros afectados automáticamente.

### Función E — Ranking ELO por deporte
**Fase 1 — Encuesta de calibración inicial**: años jugando, frecuencia, torneos, nivel autopercibido, estilo de juego. Claude asigna ELO inicial (escala 1000–2000).
**Fase 2 — Nutrición por partidos**: cada resultado registrado actualiza el ELO de ambos jugadores. Al completar 10 partidos el nivel queda "verificado".
Ranking público en la app: posición exacta entre todos los miembros del club por deporte.

### Función G — Control de Acceso Inteligente
Sistema consulta en tiempo real el estado del miembro antes de abrir el torniquete.
**⚠️ Los días de gracia y el umbral de bloqueo los define gerencia** — parámetros configurables en dashboard. No se activa sin ese acuerdo.
Si hay problema: WA proactivo antes de que el miembro llegue. Sin confrontación humana.

### Función H — Pronóstico del Clima para Canchas
API: Tomorrow.io (geo: Paseo de los Flamingos 38, Bahía de Banderas).
Todas las canchas de tenis (arcilla y dura) son abiertas — siempre afectadas por lluvia.
Cron cada 3h → si lluvia en próximas 3h → WA a miembros con reserva en cancha exterior → opción de reprogramar o cambiar.

### Función K — Ligas Permanentes
Temporadas de semanas/meses con calendario fijo, mismos jugadores, tabla de posiciones en tiempo real.
Tipos: individual, parejas fijas, mixta.
Los partidos de liga tienen prioridad en asignación de canchas. Resultados actualizan ELO. Integradas en `app.bahiaclub.mx`.
**Por qué es el mejor feature de retención**: miembro con liga activa no cancela membresía.

### Función L — Logros y Badges
Medallas digitales automáticas en el perfil de la app. Categorías: asistencia (10/100 visitas, rachas), deportivo (primer torneo, campeón, top 10 ELO), comunidad (clases, NPS), antigüedad (6 meses, 1 año, 3 años).
Notificación WA cuando se gana un logro.

### Función M — Gestión de Casilleros
Registro digital: qué casillero tiene cada miembro, vestidor, fecha asignación, llave entregada.
Al cancelar membresía → casillero pasa a "pendiente devolución" automáticamente.
Dashboard admin: grid visual por vestidor, colores por estado, lista de devoluciones pendientes.

---

## Orden de implementación

### Oleada 1 — Infraestructura base (semanas 1-3)
1. Agente 1 (check-in) — genera el dato fundamental
2. Agente 3 (cobranza) — base de membresías y pagos
3. Agente 6 (reservas) — alto valor inmediato, elimina Playtomic
4. `app.bahiaclub.mx` — plataforma de reservas (ahorra $50k MXN/mes desde día 1)

### Oleada 2 — Inteligencia operativa (semanas 4-6)
5. Agente 5 (prospección) — WA + IG/FB DMs
6. Agente 9 (NPS) — encuesta mensual inteligente
7. Agente 10 (mantenimiento) — reportes + preventivo programado
8. Agente 15 (clases) — reservas con instructor
9. Función B (day pass) — embudo $500 → membresía

### Oleada 3 — Estrategia y crecimiento (semanas 7-10)
10. Agente 4 (eventos + ligas K) — planeación con datos reales
11. Agente 7 (contabilidad) — P&L en vivo para socios
12. Agente 11 (staff) — horarios por demanda real
13. Función A (puntos) — programa de lealtad
14. Función E (ELO) — ranking por deporte
15. Función L (badges) — logros automáticos
16. Función M (casilleros) — gestión digital

### Oleada 4 — Inteligencia avanzada (semanas 11-14)
17. Agente 8 (CFO) — proyecciones + alertas tempranas
18. Agente 2 (marketing) — requiere Meta Business confirmado
19. Agente 12 (reputación) — requiere Meta Business confirmado
20. Función C (comunicaciones segmentadas)
21. Función G (control de acceso) — requiere acuerdo de gerencia sobre días de gracia
22. Función H (clima) — alertas para canchas exteriores

**Why:** La Oleada 1 genera los datos que todos los demás agentes necesitan para funcionar, y elimina el gasto de Playtomic ($50k/mes) desde el primer mes, financiando el desarrollo del resto del sistema.
**How to apply:** Implementar en este orden estrictamente. No empezar una oleada sin completar la anterior.
