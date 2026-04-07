# Finanzas Personales — Contexto del Proyecto

## Descripcion
Dashboard financiero personal de Luis Molina. Arquitectura simplificada (2 workflows n8n + 1 PWA).
Reemplaza un sistema de 9 workflows y 37 nodos que se habia vuelto inmanejable.

## Stack
- **Frontend**: PWA (HTML/CSS/JS vanilla) en Netlify
- **Backend**: n8n (2 workflows activos)
- **BD**: Supabase PostgreSQL — schema `finanzas_personales`
- **WhatsApp**: YCloud API (numero `5493815762656`)
- **IA**: OpenAI GPT-4.1 (agente n8n)

---

## URLs clave

| Recurso | URL |
|---|---|
| PWA (produccion) | https://finanzas-pwa-885.netlify.app |
| Admin Netlify | https://app.netlify.com/projects/finanzas-pwa-885 |
| GitHub repo | https://github.com/luis171190/finanzas-pwa |
| API Dashboard | https://sswebhook.myaivaro.com/webhook/dash-data |
| Webhook Asistente | https://sswebhook.myaivaro.com/webhook/asistente-financiero |

---

## Workflows n8n activos

### 1. `💰 Asistente Financiero` — ID: `MRW99xAuOv5kVtHS`
- Webhook POST `/asistente-financiero` (YCloud + Evolution API)
- 21 nodos, activo
- Tools disponibles: `registrar_gasto`, `eliminar_gasto`, `consultar_gastos_recientes`, `registrar_ingreso_extra`, `actualizar_billetera`, `consultar_resumen_completo`, `consultar_presupuesto`, `consultar_compromisos_tc`, `consultar_pago_tarjetas`, `consultar_vencimientos_proximos`, `simular_cuotas`
- Modelo: GPT-4.1

### 2. `⚙️ Motor Financiero` — ID: `QP4sGT61smMGJU2m`
- Cron diario 9am (America/Argentina/Buenos_Aires)
- 19 nodos, activo
- Ruteo por dia: **siempre** (TC + vencimientos) | **lunes** (resumen semanal) | **dia 25** (recordatorio liquidacion) | **dia 1** (cierre mensual + cuotas + mes siguiente)

### 3. `📡 API Dashboard` — ID: `E0AwewlJN2GKMDzF`
- Webhook GET `/dash-data`
- Devuelve JSON con presupuesto, billeteras, compromisos, ultimos gastos, historial
- CORS abierto (`Access-Control-Allow-Origin: *`)
- Activo — es el datasource de la PWA

---

## Credenciales n8n (IDs)
- **BD Supabase**: `c5vJmoXD1JooCpKG`
- **OpenAI**: `doxD8wCrP7dw8oHa`
- **YCloud**: `pFmujxRRv8RAE6kX`

---

## Schema BD (`finanzas_personales`)

Tablas core:
- `presupuesto_mensual` — presupuesto mes a mes (clave: `anio_mes` formato `YYYY-MM`)
- `gastos_variables` — registro de gastos del dia a dia
- `gastos_fijos` — suscripciones y pagos fijos mensuales
- `compromisos_tarjeta` — cuotas activas de tarjetas
- `billeteras` + `tipos_billetera` — saldos por billetera
- `tipo_cambio` — dolar blue diario (fuente: dolarapi.com)
- `configuracion` — regla presupuestaria (40/25/20)
- `categorias_gasto` — categorias con iconos
- `ingresos` — ingreso base mensual

Vista clave: `vista_resumen_completo` — no incluye `anio_mes`, hay que llamar `consultar_presupuesto` separado para obtener el mes.

---

## Regla presupuestaria
- 40% Necesidades
- 25% Deseos
- 20% Ahorro
- 10% Reserva Tarjetas (solo en system prompt, no en BD)
- 5% Imprevistos (solo en system prompt, no en BD)

Ingreso: U$D 1.500/mes. Mayo 2026 incluye bono U$D 800 extra (total U$D 2.300).
Cobro: ~dia 16 de cada mes (1er jueves habil empresa España, impacta ~4 dias habiles despues).

---

## Billeteras
| Nombre | Codigo BD |
|---|---|
| Fondo Juan (Emergencia) | `FONDO_JUAN` |
| Fondo para Auto | `FONDO_AUTO` |
| Dinero Operativo | `OPERATIVO` |
| Reserva Tarjetas | `RESERVA_TARJETAS` |

Transferencias van a IEB+: alias `luismolina.ieb`

---

## Tarjetas de credito
- `CREDITO_GALICIA` — principal
- `NARANJA` — secundaria (Naranja X)
- `Mastercard Black Galicia` — vieja, casi terminada
- `Mastercard Galicia` — cuotas puntuales

---

## Deploy PWA

```bash
# Desde la carpeta del proyecto:
cd "C:/Users/luis_/OneDrive/Documentos/Proyectos Personales/Proyectos Claude Code/finanzas-pwa"

# Deploy a produccion:
npx netlify-cli deploy --prod --dir=.

# Si pide login:
npx netlify-cli login
# Autorizar en navegador, luego:
npx netlify-cli login --check <ticket-id>
```

El sitio esta linkeado al account de Capa a Capa en Netlify.
Site ID: `e9349f85-ded1-4137-acb8-e428c3d4c1a5`

---

## Archivos del proyecto

```
finanzas-pwa/
├── index.html       # PWA completa (single file, ~950 lineas)
├── manifest.json    # PWA manifest (iconos, nombre, colores)
├── sw.js            # Service Worker (cache shell, network-first para API)
├── netlify.toml     # Headers de seguridad + Content-Type para manifest
└── CLAUDE.md        # Este archivo
```

---

## Bugs corregidos (2026-04-07)

1. **API Dashboard no devolia datos**: El webhook estaba en modo `onReceived` (responde vacio inmediatamente). Fix: agregar nodo `Respond to Webhook` conectado al resultado de la query.

2. **Agente inventaba el mes**: El system prompt no instruia a consultar el mes de la BD. La `vista_resumen_completo` no expone `anio_mes`. Fix: instruccion explicita de llamar `consultar_presupuesto` para obtener mes y TC. 

3. **Presupuesto Abril desactualizado**: `presupuesto_mensual` tenia U$D 1.400 en vez de U$D 1.500. Fix: UPDATE con TC actual al ejecutar.

---

## Patrones importantes

- **Temp workflows**: Para queries de BD usar patron webhook → postgres → activar → trigger → leer ejecucion → borrar. No usar `respondToWebhook` en modo test.
- **patchNodeField**: Para editar system prompts usar patches quirurgicos, nunca reemplazar todo el string.
- **DDL bloqueado**: La credencial BD Supabase no tiene permisos ALTER TABLE. Solo DML.
- **Avanzar cuotas**: Lo hace el Motor Financiero el dia 1 del mes automaticamente.
