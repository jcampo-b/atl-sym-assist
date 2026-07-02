# Voice AI — V3 Planning

> Fuente: `Voice_Caller_V3.pdf`

## Architecture Decisions

- **Twilio subaccounts por PM** (patrón oficial multi-tenant de Twilio).
- **Credenciales encriptadas at rest** en la base de datos.
- **Onboarding vía invite link único** (uso único, sin expiración).
- **Pagos manejados fuera del sistema** en V3.0 — el acceso se controla vía invite.
- **State persistence** durante el provisioning, para permitir recuperación tras un fallo.

---

## V3.0 — MVP

**Goal:** que un segundo PM pueda usar el sistema de forma independiente, sin tocar código.

**Estimado total:** ~56 hrs / ~7 días hábiles (1 dev)

| # | Área | Tarea | Detalle | Est. |
|---|---|---|---|---|
| 1 | DB | `voice_clients` table | Guarda la config, credenciales y estado de cuenta de cada PM | 2h |
| 2 | DB | `vc_invites` table | Trackea invite links: generado, reservado, usado | 2h |
| 3 | Core | `VoiceClientContext` | Objeto que mantiene la config del PM activo durante un request | 2h |
| 4 | Core | `ResolveVoiceClientMiddleware` | Identifica qué PM está siendo llamado según el número marcado | 3h |
| 5 | Core | `RentManagerHttpClient` refactor | Hace que la conexión a Rent Manager use las credenciales propias de cada PM | 8h |
| 6 | Core | `RentManagerAuthService` | Mantiene el token de sesión de Rent Manager separado por PM en cache | 2h |
| 7 | Core | `VoiceAIController` | Vincula cada llamada entrante al PM correcto antes de iniciar la sesión de IA | 4h |
| 8 | Core | `VoiceAIWebSocketHandler` | Asegura que la sesión de IA use los datos del PM correcto durante toda la llamada | 4h |
| 9 | Core | `VoiceAIService` + `TwilioClient` | Usa el teléfono de soporte y las credenciales Twilio propias de cada PM | 3h |
| 10 | Provisioning | `TwilioProvisioningService` | Crea el subaccount Twilio del PM, compra su número, registra el webhook. Guarda progreso en cada paso para no perder nada si algo falla | 10h |
| 11 | UI | Admin — Generate Invite | Página simple para que el Admin genere un invite link para compartir con el PM | 4h |
| 12 | UI | Form — Step 1 (validación) | El PM completa sus datos; el sistema valida las credenciales de Rent Manager antes de gastar nada | 6h |
| 13 | UI | Form — Step 2 (confirmación) | El PM revisa y confirma; el sistema provisiona su cuenta y muestra el número asignado | 6h |
| 14 | Migration | Migrar config existente | Mover el setup actual a la nueva estructura multi-PM sin downtime | 8h |

### Riesgos clave

- **#5 (RentManagerHttpClient refactor)**: toca el core del sistema — requiere test de regresión completo del flujo.
- **#10 (Provisioning)**: hace llamadas reales a la API de Twilio — debe probarse primero en modo test de Twilio.
- **#14 (Migration)**: debe hacerse con **cero downtime** en producción.

---

## V3.1 — Post-MVP

**Goal:** flujo pulido, self-service y visibilidad operativa.

**Estimado total:** ~24–44 hrs / ~3–5.5 días (según decisión de pagos)

| # | Área | Tarea | Detalle | Est. | Depende de |
|---|---|---|---|---|---|
| 1 | Email | Email service setup | Configurar un proveedor de email transaccional (ej. SendGrid) | 3h | Decisión del cliente |
| 2 | Email | Notification templates | Emails de "Account approved" y "Account active — your number is X" | 5h | #1 |
| 3 | Admin | `voice_clients` management panel | Listado de cuentas PM con estados, reintento de provisioning fallido, suspensión | 10h | — |
| 4 | Onboarding | Public registration (sin invite) | Cualquiera puede registrarse desde una página pública; reemplaza el flujo por invite | 6h | Decisión del cliente |
| 5 | Payments | Stripe integration | Pago automatizado antes del provisioning, con manejo completo de webhooks | 20h | Decisión del cliente |

---

## Summary

| Versión | Incluye | Estimado | Resultado |
|---|---|---|---|
| **V3.0** | Core multi-tenant + provisioning Twilio + onboarding basado en invite | ~7 días hábiles | Un segundo PM puede operar de forma independiente |
| **V3.1** | Emails + panel admin + registro público + Stripe (opcional) | ~3–5.5 días | Producto pulido, self-service, visibilidad completa |
| **Total** | | **~10–12.5 días hábiles** | |

---

## Client Validation Questions

### 💰 Money
- ¿SymAssist gana un margen sobre los costos, o los PMs solo pagan el costo de plataforma (Twilio + infra)?
- ¿El pago se maneja dentro del sistema (checkout) o fuera (factura, transferencia bancaria)?
- Si el provisioning falla después de un pago — ¿intervención humana para arreglarlo, o retry automático con reembolso si no es recuperable?
- ¿Quién absorbe el costo del número Twilio (~$1/mes por PM)? ¿Plataforma o PM directamente?

### 🎨 UX
- ¿Invite links personales o página de registro pública que cualquiera pueda encontrar?
- ¿Los PMs necesitan ver el estado de su propia cuenta (pending, active, suspended)?
- ¿Cómo se entera el PM de que su cuenta está lista y cuál es su número? (email, contacto manual, en pantalla?)
- ¿Los PMs pueden actualizar sus propias credenciales de Rent Manager, o es una operación de admin?

### 🧩 Strategy
- ¿Números Twilio manejados bajo una sola cuenta de plataforma (más simple, recomendado) o cada PM tiene su propia cuenta Twilio (aislamiento total)?
- ¿Qué pasa cuando un PM cancela o es suspendido — se libera, archiva o transfiere el número?
- Si un PM se va, ¿mantiene la propiedad del número Twilio o se queda en la plataforma?

### 📦 Product
- ¿El saludo del IVR es personalizable por PM (nombre del negocio, mensaje de bienvenida) o el mismo script sirve para todos?
- ¿La transferencia a un humano es obligatoria desde el día uno, u opcional por PM?
- ¿Qué información se necesita del PM en el registro además de las credenciales de Rent Manager? (nombre de contacto, email de facturación, nombre de la compañía, número de redirección humana)
- ¿El sistema debe soportar múltiples idiomas por PM, o solo inglés por ahora es suficiente?

### ⚙️ Operations
- ¿Cuántos PMs se esperan en la primera ola, y cuál es el objetivo de crecimiento para el año uno?
- ¿Hay un SLA específico de onboarding? (ej. "la cuenta debe estar activa dentro de X minutos desde el envío del formulario")
- ¿Cada IA de PM necesita conocer el nombre específico de su negocio, o es aceptable una configuración genérica?

---

## Answers — Client Validation Questions
### (Doug, Daily Sync — 25 jun 2026)

> Capturado de la llamada con Doug el 2026-06-25. Los ítems marcados **PENDING** no se cubrieron todavía; Doug traerá la sección de Product la semana próxima.

### 💰 Money

- **¿Margen o solo pass-through de costo?**
  Doug: SymAssist gana un margen. El precio por unidad debe cubrir todos los costos operativos **más** ganancia.

- **¿Pago dentro del sistema o externo?**
  Doug: **Externo**, manejado en **Zoho** (pagos nativos + reconciliación bancaria automatizada). El billing es SymAssist-a-PM directamente; Rent Manager no es el merchant of record.

- **¿Provisioning falla después del pago — fix manual o retry/refund automático?**
  **PENDING** (no discutido).

- **¿Quién absorbe el costo de ~$1/mes del número Twilio?**
  Doug: La plataforma lo absorbe y lo recupera a través de la tarifa mensual por unidad.

### 🎨 UX

- **¿Invite links personales o página de registro pública?**
  Doug: **Ninguna** de las dos — el PM hace clic en "Request Activation" dentro de Rent Manager y luego recibe un link de sign-up de SymAssist. V3.0 se mantiene basado en request, sin página pública.

- **¿Los PMs ven el estado de su propia cuenta (pending/active/suspended)?**
  Doug: **No.**

- **¿Cómo se entera el PM de que la cuenta está lista y cuál es su número?**
  Doug: Por **email**.

- **¿Los PMs pueden auto-actualizar sus credenciales de Rent Manager?**
  Doug: **No** — es una operación de admin. Con el partnership/partner token de RM, probablemente ni haga falta recolectar la API key del PM.

### 🧩 Strategy

- **¿Una cuenta Twilio de plataforma o una por PM?**
  Doug: **Una sola cuenta de plataforma.** Los PMs ni siquiera van a saber la palabra "Twilio".

- **¿Al cancelar/suspender — el número se libera, archiva o transfiere?**
  Doug: Se resuelve vía **forwarding** — se redirige el número de negocio existente del PM hacia el número Twilio de la plataforma, así que cancelar simplemente implica dejar de redirigir (inmediato).

- **¿Si un PM se va, quién es dueño del número Twilio?**
  Doug: Se queda con la plataforma. El PM conserva su propio número legacy, que solo estaba siendo redirigido.

### 📦 Product

- **¿Saludo del IVR personalizable por PM?**
  **PENDING** — Doug confirma la semana próxima.

- **¿Transferencia a humano obligatoria desde el día uno?**
  **PENDING** — Doug confirma la semana próxima.

- **¿Qué info se necesita del PM al registrarse, además de credenciales RM?**
  Mayormente cubierto por la activación de RM (nombre de contacto, teléfono y email vienen incluidos); con el partner token la API key no se necesita del PM. No hay una lista final de campos extra.

- **¿Múltiples idiomas por PM o solo inglés?**
  **Solo inglés.**

### ⚙️ Operations

- **¿PMs en la primera ola / objetivo de crecimiento año uno?**
  Doug: ~10 design partners en los primeros meses, luego 1-2 por semana. Una parte significativa de los primeros ~50 clientes se onboardeará gratis.

- **¿SLA de onboarding?**
  Doug: Sin SLA estricto en minutos; el objetivo es que la cuenta esté lista dentro de ~24h, excluyendo el paso de forwarding del número telefónico.

- **¿Nombre de negocio por PM en la IA o config genérica?**
  **PENDING** (parte de la sección Product que quedó pendiente).
