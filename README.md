# Incident-Replay.-PBG
# Incident Replay — Centro de Comando de Incidentes (Reto PBG)

Herramienta web (un solo `index.html`, sin backend ni dependencias) para reconstruir un incidente en una plataforma de cotización de seguros de vida operada por un agente de IA.

## Cómo correrlo
Abre `index.html` en el navegador, o desplégalo tal cual en Vercel/Netlify/GitHub Pages.

## Qué hace
- Parsea `server.log`, `carrier.log` y `deploy.log`, y normaliza todo a UTC (`deploy.log` viene en -05:00).
- Línea de tiempo con scrubber: grilla de 20 asientos, estado del servidor y eventos correlacionados.
- Ledger de evidencia: **Sé** (con archivo y línea), **Infiero** (con confianza) y **Falta verificar** (con comando).
- Ranking de hipótesis, sección "Por qué NO es esto", siguiente paso seguro y mensaje para el asesor.

## Decisiones
- Reglas y conclusiones deterministas; un LLM solo redactaría, nunca decide la causa.
- Sin backend: cero riesgo de falla en demo.
- **Incidentes concurrentes, no anidados**:
  - **WebSocket (S003/S007/S014)** — impacto a usuario. Arrancó **ANTES** del deploy (13:03:00Z). Es la causa raíz del impacto.
  - **Deploy parcial .29.5** — crash loop server-side, concurrente. **Sin impacto a usuario demostrado**: Alpha cotizó 200 en pleno crash loop y /health respondió 200 (vía Cloudflare).
  - **Beta 302→401** — auth, independiente, previo al deploy.
- **No culpamos a `normalizeCallIndex`**: la evidencia lo sitúa 2 minutos DESPUÉS de la pérdida de heartbeat. Causalidad invertida.

## Jerarquía de hipótesis
1. **H1 — WebSocket (70%)**: degradación de nodo compartido por S003/S007/S014. Causa del impacto a usuario.
2. **H2 — Beta auth (65%)**: el agente no conserva sesión tras el 302. Independiente.
3. **H3 — Deploy crash loop (75%)**: hecho confirmado, pero **no es la causa del WebSocket**. Incidente concurrente a frenar.

## Siguiente paso seguro (en orden)
1. No reiniciar producción con S002/S005/S011 en llamada activa.
2. Verificar nodo WebSocket de S003/S007/S014 (read-only).
3. Congelar deploy .29.5 — no promover.
4. Verificar hash de `call-index.js`; rollback a .29.4 si está roto.
5. Validar login Beta en staging.
6. Prevención: smoke test de imports en preflight.

## Limitaciones
- Los datos de ejemplo están precargados; no hay carga de logs propios.
- Las conclusiones y porcentajes de confianza están escritos a mano, no calculados por un motor de reglas. El diseño lo permite, pero el prototipo lo simplifica.
- Sin tests automatizados.
- Impacto en cotizaciones a Beta: desconocido (requiere logs del agente).

## Datos
Todos los datos son sintéticos (carpeta `data/`).
