# Power: `deteccion-fuerza-bruta`

Power **defensivo** (Blue Team) que analiza capturas de `tcpdump` y logs de
autenticación para **detectar** ataques de fuerza bruta contra
formularios/paneles de login, distinguirlos de otras fallas de red, y
verificar si las medidas de prevención/hardening estaban en su lugar.

- **Tipo:** `defensive` (no ofensivo). `ecCouncilPhase: null`, conforme a lo
  solicitado — ver nota sobre el contrato `x-ethack` más abajo.
- **Herramientas reales:** `tcpdump` (lectura de capturas) + análisis de
  logs con `grep`/`awk`. **No usa Hydra ni ninguna herramienta de ataque.**
- **Equipo / responsable:** Blue Team — Diego Salinas (`owner: diego-salinas`).

> **Nota sobre el contrato `x-ethack`:** el modelo estándar de Power (ver
> `powers/_template/README.md`, regla 2, y `scripts/power_adapter.py`) pide
> que `ecCouncilPhase` y `forensicsStage` sean mutuamente excluyentes (XOR):
> exactamente uno de los dos debe declarar la ubicación del Power en el
> ciclo. `forensicsStage` acepta `Detect|Isolate|Collect|Preserve|Analyze|Report`
> (Flujo_Investigacion; ver uso real en `dfir-linux/POWER.md`, que declara
> `"Analyze"`, y `network-forensics/POWER.md`, que declara `"Collect"`).
>
> Este Power es de **detección** (Blue Team), no ofensivo, así que se deja
> `ecCouncilPhase: null` y se asigna **`forensicsStage: "Detect"`** — la
> primera etapa del Flujo_Investigacion — para satisfacer la regla XOR y
> pasar `make validate`. Esta asignación es una aproximación práctica: el
> Power no sigue el resto del Flujo_Investigacion forense (`Isolate`,
> `Collect`, `Preserve`, `Analyze`, `Report`), solo cubre el momento de
> detectar el patrón de fuerza bruta en logs/capturas ya recolectados.
> **Queda a criterio del Líder Técnico** decidir si conviene mantener esta
> asignación o si el contrato debe incorporar una categoría propia para
> Blue Team (análoga a `track: GRC|IR` que ya usan `grc-nist-csf` e
> `incident-handling-nist`) en lugar de forzar `forensicsStage`.

## Overview — de qué apunte sale este Power

Este Power se destiló del apunte `docs/fuentes/896492b3-4_fuerza_bruta_ejercicio.pdf`
("Fuerza bruta y sitio web comprometido", práctica de REDES · INCIDENTES).

El apunte es, en su totalidad, un **ejercicio forense/defensivo**: enseña a
reconocer en `tcpdump` un sitio que redirige a sus visitantes a una página
falsa, qué es un ataque de fuerza bruta, cómo se ve ese ataque en un log de
autenticación (aunque la captura del ejercicio no lo muestre directamente),
cómo distinguir el caso de otras fallas de red, y cómo se previene. Es
exactamente el lado de **detección** del mismo incidente que motivó el
Power ofensivo `fuerza-bruta-login` — aquí se trabaja desde la perspectiva
del analista, no del atacante.

Caso del apunte, resumido (Content was rephrased for compliance with
licensing restrictions): un exempleado prueba contraseñas por defecto
conocidas contra el panel de administración de `yummyrecipesforme.com` y
entra porque la contraseña de admin nunca se cambió y no había límite de
intentos. Incrusta JavaScript malicioso, cambia la contraseña de admin, y
los visitantes terminan redirigidos a un dominio falso
(`greatrecipesforme.com`, typosquatting). El analista lo reproduce en un
sandbox con `tcpdump` y confirma la redirección a partir de las quejas de
los clientes.

## Reglas obligatorias (origen: el apunte)

1. **El patrón `FAILED`→`SUCCESS` desde una misma IP** (apunte, sección
   "Cómo se vería una fuerza bruta (no viene en tu log)"): en los registros
   del servidor, una fuerza bruta se ve como muchos intentos fallidos
   seguidos, desde la misma IP, contra la misma cuenta, y al final uno
   exitoso. Ejemplo citado en el apunte:
   ```
   10:02:11 FAILED login user=admin from 198.51.100.44
   10:02:12 FAILED login user=admin from 198.51.100.44
   10:02:12 FAILED login user=admin from 198.51.100.44
   ... cientos más ...
   10:09:47 SUCCESS login user=admin from 198.51.100.44
   ```
   El apunte aclara que esta vista es del lado del **servidor**; la
   captura de `tcpdump` del ejercicio es del lado del **visitante**, por lo
   que no muestra el ataque al panel directamente — solo sus consecuencias.
2. **Las 4 capas de TCP/IP** (apunte, sección "¿Qué protocolo está
   involucrado? Las 4 capas de TCP/IP"): el Power debe ubicar cada
   protocolo observado en su capa antes de señalar cuál "carga el
   problema":
   - *4. Aplicación*: HTTP (`.http`, puerto 80, `GET / HTTP/1.1`) y DNS
     (`.domain`, puerto 53, `A?`). Aquí vive la respuesta.
   - *3. Transporte*: TCP (`Flags [S]`, `[S.]`, `[.]`, `[P.]`) para HTTP, y
     UDP para DNS.
   - *2. Internet*: IP — la palabra `IP` al inicio de cada línea de
     `tcpdump`.
   - *1. Acceso a la red*: Ethernet/Wi-Fi — `tcpdump` no lo muestra en
     estas líneas.
   - En el caso del apunte, el protocolo que carga el problema es **HTTP**:
     es el que se usó para pedir la página comprometida, descargar el
     malware y, después, cargar el sitio falso. DNS participa, pero solo
     resuelve nombres; funcionó como debía.
3. **La regla de typosquatting** (apunte, sección "Cómo distinguir este
   caso de los anteriores", "Regla de oro de este caso"): si el log se ve
   "sano" pero el navegador termina en un dominio diferente, sospechar de
   redirección maliciosa. Buscar una **segunda consulta DNS**, sin que el
   usuario la pidiera, hacia un nombre parecido al original. En el caso del
   apunte: nadie escribió `greatrecipesforme.com`, lo pidió el malware que
   se acababa de ejecutar — dominio nuevo, casi idéntico al original, con
   otra IP.
4. **Las medidas de prevención** (apunte, sección "Cómo prevenir la fuerza
   bruta"): el Power debe verificar/recomendar, en este orden de fuerza
   según el apunte:
   - **2FA**: la más fuerte, porque sigue protegiendo incluso si la
     contraseña se adivina o se filtra.
   - **Limitar los intentos**: corta la fuerza bruta de raíz (segunda
     opción recomendada).
   - Contraseñas seguras (no de fábrica), monitorear inicios de sesión,
     cambios frecuentes de contraseña, no reutilizar contraseñas — cada una
     con su "¿habría frenado este ataque?" evaluado en el apunte.
5. **Guardia de alcance**: todo análisis se circunscribe a los activos
   declarados en `scope.yaml#assets` del engagement
   (`allowlistValidation.required: true`); este Power **solo detecta y
   reporta**, no ejecuta ninguna prueba de fuerza bruta ni interactúa
   ofensivamente con el objetivo.

## Guías de `steering/`

| Guía | Tema | Se carga cuando |
|---|---|---|
| `patron-fuerza-bruta-logs.md` | Identificar el patrón `FAILED`→`SUCCESS` desde una misma IP en logs de autenticación | Al revisar logs de autenticación de un servicio sospechoso |
| `lectura-tcpdump-capas.md` | Leer una captura `tcpdump` ubicando cada protocolo en su capa TCP/IP (Aplicación/Transporte/Internet/Acceso a la red) | Al analizar una captura de red del incidente |
| `distincion-casos.md` | Distinguir DNS caído / SYN flood / sitio comprometido (fuerza bruta + redirección) a partir de las mismas preguntas guía | Al clasificar qué tipo de incidente se está viendo |
| `prevencion-hardening.md` | Medidas de prevención (2FA, límite de intentos, etc.) y su efectividad evaluada contra el caso del apunte | Al recomendar hardening tras confirmar el hallazgo |

## Flujo de detección (según el apunte, adaptado a este Power)

1. **Recolectar evidencia**: captura `tcpdump`/`tshark` del tráfico del
   sandbox o del segmento afectado, y logs de autenticación del servicio
   (panel/formulario de login).
2. **Leer la captura por capas** (`steering/lectura-tcpdump-capas.md`):
   ubicar DNS, TCP/HTTP y detectar si hay una segunda resolución DNS no
   solicitada hacia un dominio parecido al legítimo.
3. **Buscar el patrón de fuerza bruta en los logs**
   (`steering/patron-fuerza-bruta-logs.md`): múltiples `FAILED` seguidos de
   un `SUCCESS`, misma IP, misma cuenta.
4. **Clasificar el caso** (`steering/distincion-casos.md`): confirmar que
   no se trata de un DNS caído ni de un SYN flood, usando las mismas
   preguntas guía del apunte (¿hay errores en el log?, ¿se completa el
   handshake?, ¿qué llama la atención?, ¿qué protocolo carga el problema?,
   ¿qué falló?).
5. **Verificar medidas de prevención** (`steering/prevencion-hardening.md`):
   confirmar cuáles de las medidas del apunte estaban o no activas, y
   recomendar las que habrían frenado el ataque (2FA y límite de intentos,
   según el apunte).
6. **Reportar el hallazgo**: documentar la cadena completa (acceso por
   fuerza bruta → modificación del sitio → redirección) en el
   `producesActionLog`, para consumo de Digital Forensics / Incident
   Responder.

## Modos de ejecución (`Modo_Ejecucion`)

Definidos en `plugin.json#x-ethack.executionMode` (contrato estándar del
proyecto, no viene del apunte):

| Modo | Comportamiento esperado |
|---|---|
| `script-kiddie` (default) | Aplica únicamente las búsquedas de patrón más directas del apunte (grep de `FAILED`/`SUCCESS`, verificación de una segunda consulta DNS) sobre los logs/captura ya recolectados. |
| `green-hat` | Añade la lectura completa por capas TCP/IP y la tabla de distinción de casos, documentando cada paso. |
| `pro` | Permite ajustar manualmente los filtros de `tcpdump`/`grep`/`awk` y correlacionar múltiples fuentes de log, bajo la responsabilidad del analista de no alterar la evidencia original. |

Valores fuera de `{script-kiddie, green-hat, pro}` se rechazan con
`InvalidModeError` (contrato estándar del modelo de Power).

## Prompts de arranque

- *"Analiza este log de autenticación y dime si hay un patrón de fuerza
  bruta (FAILED repetidos seguidos de SUCCESS desde la misma IP)."*
- *"Lee esta captura de tcpdump por capas y dime si hay una segunda consulta
  DNS no solicitada hacia un dominio parecido al legítimo."*
- *"Con estas tres evidencias, clasifica si el incidente es DNS caído, SYN
  flood o sitio comprometido por fuerza bruta."*
- *"Dado que el ataque tuvo éxito por contraseña de fábrica sin límite de
  intentos, ¿qué medida de prevención recomiendas primero y por qué?"*

## Artefactos de documentación (`docs/`)

Conforme al contrato `x-ethack.documentation`: `docs/prompt.md`,
`docs/sessions/`, `docs/feedback.md`. La fuente original del apunte vive en
`docs/fuentes/896492b3-4_fuerza_bruta_ejercicio.pdf`.
