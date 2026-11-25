# Prompt de generación del Power — `fuerza-bruta-login`

> Artefacto_Documentacion obligatorio. Documenta el o los prompts con los que
> generaste/iteraste este Power con Kiro. Es parte del contrato del modelo
> estándar (`x-ethack.documentation.required` incluye `prompt.md`).

## Objetivo del Power

Envuelve **Hydra** (`binary: hydra`) para probar credenciales contra un
servicio de autenticación (formulario/panel de login) dentro del alcance
autorizado del engagement. Cubre la fase EC-Council **Gaining Access**
(`ecCouncilPhase: "Gaining Access"`, `forensicsStage: null`) — la fuerza
bruta de login es acceso, no reconocimiento ni escaneo.

**Modelo usado en la generación:** Claude Opus 4.8 (vía Kiro, sesión de
agente autónomo).

**Fuente:** `docs/fuentes/4_fuerza_bruta_ejercicio.pdf` ("Fuerza bruta y
sitio web comprometido" — apunte de REDES · INCIDENTES).

## Prompt(s) usados

Prompt 1 — generación inicial del Power:

```
Usa el apunte de fuerza bruta adjunto como fuente de conocimiento. Destila su metodología,
herramientas, comandos y reglas, y completa el Power powers/fuerza-bruta-login/ partiendo de
la plantilla:

(1) Rellena los TODO del plugin.json: name: "fuerza-bruta-login", author = Diego Salinas,
owner = diego-salinas, ecCouncilPhase: "Gaining Access" (no Scanning — la fuerza bruta de
login es acceso), tools = Hydra (binary: "hydra"). Conserva allowlistValidation y
executionMode.

(2) Escribe el README.md del Power con: overview (de qué apunte sale), reglas obligatorias
citando su origen en el apunte, tabla de guías steering/, flujo del ataque, los 3 modos de
ejecución y prompts de arranque.

(3) Crea las guías steering/ por tema (p. ej. preparación del diccionario, ejecución de Hydra
contra el formulario, interpretación de resultados, límites anti-DoS).

No inventes comandos que no estén en el apunte; lo que agregues márcalo como (añadido).
Respeta la guardia de alcance.
```

El apunte se adjuntó a continuación como referencia de archivo
(`#[[file:.../4_fuerza_bruta_ejercicio.pdf]]`), extraído con `pdftotext -layout`
porque el contenido inicial llegó como PDF binario sin procesar.

## Decisiones de diseño

- **Modos de ejecución**: los 3 estándar del proyecto
  (`script-kiddie`/`green-hat`/`pro`), con `script-kiddie` como default.
  `script-kiddie` acota el diccionario a credenciales de fábrica/comunes y
  concurrencia mínima; `green-hat` amplía diccionario y aplica límites
  anti-DoS documentados; `pro` permite ajuste manual bajo responsabilidad
  del operador.
- **Allowlist**: se conservó `allowlistValidation.required: true` apuntando
  a `scope.yaml#assets`; ninguna ejecución de Hydra procede sin validar el
  objetivo primero.
- **Skills/steering añadidos**: 4 guías en `steering/` —
  `preparacion-diccionario.md`, `ejecucion-hydra-formulario.md`,
  `interpretacion-resultados.md`, `limites-anti-dos.md` — cada una con una
  sección "Qué dice el apunte" (contenido literal/citado) y una sección de
  comandos marcada `(añadido)` porque el apunte no trae sintaxis de
  herramienta.
- **Hallazgo clave de esta generación**: el apunte adjunto resultó ser un
  ejercicio **forense** (lectura de `tcpdump` de un incidente), no un
  tutorial operativo de Hydra. Esto forzó la decisión de separar
  explícitamente "lo que dice el apunte" de "lo añadido" en cada guía — ver
  `feedback.md` para el detalle de esa realimentación.
