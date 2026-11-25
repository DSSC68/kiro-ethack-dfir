# Prompt de generación del Power — `deteccion-fuerza-bruta`

> Artefacto_Documentacion obligatorio. Documenta el o los prompts con los que
> generaste/iteraste este Power con Kiro. Es parte del contrato del modelo
> estándar (`x-ethack.documentation.required` incluye `prompt.md`).

## Objetivo del Power

Power **defensivo** (Blue Team) que envuelve `tcpdump` + análisis de logs
(`grep`/`awk`) para **detectar** ataques de fuerza bruta contra
formularios/paneles de login a partir de capturas de red y logs de
autenticación. `kind: "defensive"`, `team: "blue-team"`,
`ecCouncilPhase: null`, `forensicsStage: "Detect"` (etapa del
Flujo_Investigacion asignada para cumplir la regla XOR del contrato
`x-ethack`).

**Modelo usado en la generación:** Claude Opus 4.8 (vía Kiro, sesión de
agente autónomo).

**Fuente:** `docs/fuentes/4_fuerza_bruta_ejercicio.pdf` ("Fuerza bruta y
sitio web comprometido" — apunte de REDES · INCIDENTES), el mismo apunte
usado para `fuerza-bruta-login`, releído aquí desde la perspectiva de
detección en vez de ejecución.

## Prompt(s) usados

Prompt 1 — generación inicial del Power de detección:

```
Usa el apunte adjunto como fuente. Es un ejercicio forense/defensivo: cómo detectar fuerza
bruta en logs (tcpdump + logs de autenticación) y cómo prevenirla. Completa el Power
powers/deteccion-fuerza-bruta/ como un Power de DETECCIÓN (Blue Team):

(1) plugin.json: name: "deteccion-fuerza-bruta", author = Diego Salinas, owner =
diego-salinas, kind: "defensive", team: "blue-team", ecCouncilPhase: null. tools = tcpdump +
análisis de logs (grep/awk), NO Hydra. Conserva allowlistValidation y executionMode.

(2) README.md: overview, reglas obligatorias citando su sección del apunte (el patrón
FAILED→SUCCESS desde una misma IP, las 4 capas TCP/IP, la regla de typosquatting, las medidas
de prevención), tabla de guías steering/, flujo de detección, los 3 modos de ejecución y
prompts de arranque.

(3) Guías steering/ por tema: identificar el patrón de fuerza bruta en logs, leer tcpdump por
capas, distinguir casos (DNS caído / SYN flood / sitio comprometido), y medidas de
prevención/hardening.

Todo debe salir del apunte; lo que agregues márcalo (añadido). Respeta la guardia de alcance.
```

Prompt 2 — corrección del contrato `x-ethack` (regla XOR):

```
Revisa qué valores válidos acepta forensicsStage (mira el esquema del _template,
power_adapter.py o un power forense existente como dfir-linux/digital-forensics). Como este
Power es de detección, asígnale la etapa que corresponda a "detección" (p. ej. la primera del
Flujo_Investigacion, Detect), dejando ecCouncilPhase: null, para cumplir la regla XOR. Luego
corre make validate y dime si queda en VERDE. Deja anotado en el README que se asignó esa
etapa para satisfacer el contrato, y que el Líder Técnico decida si conviene agregar una
categoría Blue Team.
```

## Decisiones de diseño

- **Modos de ejecución**: los 3 estándar del proyecto
  (`script-kiddie`/`green-hat`/`pro`), con `script-kiddie` como default.
  `script-kiddie` aplica solo las búsquedas de patrón más directas
  (`grep` de `FAILED`/`SUCCESS`, verificación de segunda consulta DNS);
  `green-hat` añade lectura completa por capas TCP/IP y la tabla de
  distinción de casos; `pro` permite ajuste manual de filtros y
  correlación de múltiples fuentes de log.
- **Allowlist**: se conservó `allowlistValidation.required: true`; el
  análisis se circunscribe a los activos declarados en
  `scope.yaml#assets`. El Power solo detecta y reporta, no ejecuta ninguna
  prueba ofensiva.
- **Skills/steering añadidos**: 4 guías en `steering/` —
  `patron-fuerza-bruta-logs.md`, `lectura-tcpdump-capas.md`,
  `distincion-casos.md`, `prevencion-hardening.md` — con las tablas y
  ejemplos citados literalmente del apunte, y comandos de `grep`/`awk`/
  `tcpdump` marcados `(añadido)` porque el apunte no trae sintaxis de
  herramienta.
- **Corrección del contrato `x-ethack` (XOR)**: la primera versión dejó
  `ecCouncilPhase: null` y `forensicsStage: null` simultáneamente, violando
  la regla XOR del modelo de Power. Se revisó `power_adapter.py` y
  Powers forenses existentes (`dfir-linux` → `"Analyze"`,
  `network-forensics` → `"Collect"`) para confirmar el enum válido
  (`Detect|Isolate|Collect|Preserve|Analyze|Report`) y se asignó
  `forensicsStage: "Detect"` — la primera etapa del Flujo_Investigacion,
  coherente con que este Power cubre el momento de detectar el patrón, no
  el resto del flujo forense. Ver `feedback.md` para el detalle de esta
  corrección y la nota dejada para el Líder Técnico.
