# Realimentación del resultado — `deteccion-fuerza-bruta`

> Artefacto_Documentacion obligatorio. Registra la realimentación del resultado
> del Power al chat/contexto de Kiro: qué salió bien, qué corregiste y qué
> aprendiste al probarlo contra el Entorno_Laboratorio.

## Qué funcionó

- **Se reorientó el mismo apunte hacia detección Blue Team** y, a
  diferencia del Power ofensivo hermano, aquí **casi todo el contenido sale
  directamente del apunte** en vez de quedar marcado `(añadido)`, porque el
  apunte es en sí mismo un ejercicio de detección/forense:
  - el patrón `FAILED`→`SUCCESS` desde una misma IP y cuenta (sección "Cómo
    se vería una fuerza bruta"), citado literalmente en
    `steering/patron-fuerza-bruta-logs.md`;
  - las 4 capas de TCP/IP con la lectura línea por línea del log de
    `tcpdump` del caso (sección "¿Qué protocolo está involucrado?"),
    trasladadas completas a `steering/lectura-tcpdump-capas.md`;
  - la regla de oro de typosquatting (segunda consulta DNS no solicitada
    hacia un dominio parecido), documentada en `steering/distincion-casos.md`
    junto con la tabla comparativa DNS caído / SYN flood / sitio
    comprometido;
  - las medidas de prevención con su tabla de efectividad evaluada
    (2FA primero, límite de intentos segundo), en
    `steering/prevencion-hardening.md`.
- El `README.md` cita la sección exacta del apunte para cada regla
  obligatoria, lo que permite auditar de un vistazo qué viene de la fuente.

## Qué se corrigió tras la realimentación

- **Los comandos de `tcpdump`/`grep`/`awk` van marcados `(añadido)`.** El
  apunte explica el *qué* y el *por qué* (qué patrón buscar, qué capa mirar)
  pero no da sintaxis de herramienta para producir esos filtros sobre una
  captura o un log reales; esa sintaxis se agregó aparte en cada guía de
  `steering/` y se etiquetó como tal, siguiendo la misma disciplina que en
  `fuerza-bruta-login`.
- **Arreglo del XOR del contrato `x-ethack`.** La primera versión dejó
  `ecCouncilPhase: null` y `forensicsStage: null` simultáneamente (siguiendo
  la instrucción inicial al pie de la letra), lo que rompía la regla XOR
  del modelo de Power (`ecCouncilPhase` **o** `forensicsStage`, exactamente
  uno). Se revisó `power_adapter.py` y el enum documentado en
  `powers/_template/README.md`
  (`Detect|Isolate|Collect|Preserve|Analyze|Report`), y se contrastó contra
  usos reales en Powers forenses (`dfir-linux` → `"Analyze"`,
  `network-forensics` → `"Collect"`). Se asignó **`forensicsStage: "Detect"`**
  — la primera etapa del Flujo_Investigacion — porque es la que corresponde
  al momento que cubre este Power (detectar el patrón en logs/capturas ya
  recolectados, sin aislar, recolectar, preservar, analizar en profundidad
  ni reportar formalmente).
- Esta asignación se dejó anotada explícitamente en el `README.md` como una
  aproximación práctica para pasar `make validate`, **no como una
  afirmación de que el Power cubre todo el Flujo_Investigacion forense**, y
  se deja a criterio del Líder Técnico si prefiere mantenerla o si el
  contrato debería incorporar una categoría propia para Blue Team (análoga
  a `track: GRC|IR`, que ya usan `grc-nist-csf` e `incident-handling-nist`
  para Powers que tampoco caen limpiamente en fase EC-Council ni etapa
  forense).

## Pendientes / mejoras futuras

- Confirmar con el Líder Técnico si `forensicsStage: "Detect"` se queda
  como solución definitiva o si se introduce un `track` tipo `"BlueTeam"`
  en el contrato `x-ethack` para Powers de detección que no son forenses
  completos.
- Probar las guías de `steering/` contra un log de autenticación y una
  captura `tcpdump` reales del laboratorio (no solo el ejemplo ilustrativo
  del apunte), para confirmar que los comandos `(añadido)` de `grep`/`awk`
  producen el resultado esperado sobre datos reales.
