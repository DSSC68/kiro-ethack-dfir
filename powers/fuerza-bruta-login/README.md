# Power: `fuerza-bruta-login`

Power de **Gaining Access** que envuelve **Hydra** para probar credenciales
contra un servicio de autenticación (panel/formulario de login) dentro del
alcance autorizado del engagement (`scope.yaml#assets`).

- **Fase EC-Council:** `Gaining Access` (la fuerza bruta de login es acceso,
  no reconocimiento ni escaneo).
- **Herramienta real:** Hydra (`binary: hydra`).
- **Equipo / responsable:** Red Team — Diego Salinas (`owner: diego-salinas`).

## Overview — de qué apunte sale este Power

Este Power se destiló del apunte `docs/fuentes/896492b3-4_fuerza_bruta_ejercicio.pdf`
("Fuerza bruta y sitio web comprometido", práctica de REDES · INCIDENTES).

Importante sobre el alcance del apunte: es un **ejercicio forense de red**
(lectura de `tcpdump`) que usa un ataque de fuerza bruta como *causa raíz* de
un incidente — un sitio comprometido que redirige a sus visitantes a una
página falsa. El apunte explica **qué es** la fuerza bruta, **cómo se ve**
en un log de autenticación, y **cómo se previene**, pero **no incluye ningún
comando de Hydra ni de otra herramienta de ejecución** de fuerza bruta. Esa
pieza operativa (comandos de Hydra) se agrega en este Power y se marca
explícitamente como **(añadido)** en las guías de `steering/`, conforme a la
guardia de alcance de este proyecto: no se inventan comandos atribuyéndolos
al apunte.

Caso del apunte, resumido (Content was rephrased for compliance with
licensing restrictions): un exempleado prueba contraseñas por defecto
conocidas contra el panel de administración de `yummyrecipesforme.com`.
Entra porque la contraseña de admin nunca se cambió y no había límite de
intentos. Una vez dentro, inyecta JavaScript malicioso, cambia la contraseña
de admin para sacar al dueño legítimo, y los visitantes terminan
redirigidos a un dominio falso (`greatrecipesforme.com`) por typosquatting.

## Reglas obligatorias (origen: el apunte)

1. **Fallas que habilitan la fuerza bruta** (apunte, sección "Qué pasó" /
   "Qué es un ataque de fuerza bruta"): contraseña por defecto sin cambiar +
   ausencia de límite de intentos, 2FA o monitoreo de inicios de sesión. El
   Power **solo tiene sentido como prueba de estas dos fallas**; no se usa
   para "romper" una contraseña fuerte y correctamente protegida.
2. **Taxonomía de ataques** (apunte, sección 02): el Power debe declarar qué
   tipo de ataque ejecuta —
   - *Fuerza bruta simple*: combinaciones exhaustivas (lento, útil solo con
     contraseñas cortas).
   - *Ataque de diccionario*: contraseñas comunes o por defecto (`admin`,
     `123456`, `password`). Es el que ocurrió en el caso del apunte.
   - *Credential stuffing*: usuario+contraseña filtrados de otro sitio.
   - *Password spraying*: una misma contraseña contra muchas cuentas, para
     no activar bloqueos.
3. **Cómo se ve en el log** (apunte, sección "Cómo se vería una fuerza bruta"):
   muchos intentos `FAILED login` seguidos, desde la misma IP, contra la
   misma cuenta, y al final un `SUCCESS`. Un resultado de este Power debe
   poder explicarse con ese mismo patrón al Blue Team / Digital Forensics
   (`producesActionLog: true`).
4. **Protocolo involucrado** (apunte, sección 03): HTTP (capa de Aplicación)
   es el protocolo típico de un formulario de login web; TCP lo transporta.
   Este Power asume un objetivo HTTP/HTTPS o un servicio de autenticación de
   red equivalente (SSH, FTP, etc., soportado por Hydra), siempre dentro del
   alcance.
5. **Regla de oro de typosquatting post-acceso** (apunte, sección 05): si
   tras obtener acceso aparece una segunda consulta DNS no solicitada hacia
   un dominio parecido al legítimo, es señal de que el acceso ya fue abusado
   para modificar el sitio. Esto es responsabilidad de Maintaining
   Access/Covering Tracks, **no** de este Power — este Power se detiene al
   confirmar la credencial válida.
6. **Guardia de alcance**: toda ejecución valida el objetivo contra
   `scope.yaml#assets` del engagement (`allowlistValidation.required: true`)
   antes de lanzar Hydra. Si el objetivo no está en la allowlist, el Power
   rechaza la ejecución.
7. **Límites anti-DoS (origen: el apunte, sección 06 "Cómo prevenir")**: el
   apunte lista `Limitar los intentos` como la medida que "corta la fuerza
   bruta de raíz". Un operador responsable de este Power debe limitar su
   propia velocidad/concurrencia para no convertir la prueba en una negación
   de servicio contra el objetivo autorizado — ver
   `steering/limites-anti-dos.md`.

## Guías de `steering/`

| Guía | Tema | Se carga cuando |
|---|---|---|
| `preparacion-diccionario.md` | Construcción del diccionario de usuarios/contraseñas (de fábrica, comunes) | Antes de lanzar Hydra, al definir qué credenciales probar |
| `ejecucion-hydra-formulario.md` | Comando(s) de Hydra contra un formulario/panel de login HTTP, y contra servicios de red | Al ejecutar el ataque |
| `interpretacion-resultados.md` | Cómo leer la salida de Hydra y mapearla al patrón de log `FAILED`/`SUCCESS` del apunte | Al revisar resultados y documentar el hallazgo |
| `limites-anti-dos.md` | Límites de velocidad/concurrencia para no provocar denegación de servicio | Antes y durante la ejecución, en cualquier modo |

## Flujo del ataque (según el apunte, adaptado a este Power)

1. **Reconocimiento previo** (fuera de este Power): confirmar que el
   objetivo expone un panel/formulario de login y que está en la allowlist.
2. **Preparación del diccionario** (`steering/preparacion-diccionario.md`):
   priorizar credenciales de fábrica y contraseñas comunes — el apunte señala
   que así se produjo el acceso real en el caso (ataque de diccionario).
3. **Ejecución de Hydra** (`steering/ejecucion-hydra-formulario.md`):
   lanzar el ataque respetando los límites anti-DoS.
4. **Interpretación de resultados** (`steering/interpretacion-resultados.md`):
   confirmar la credencial válida y registrar el hallazgo en el
   `producesActionLog` para Blue Team / Digital Forensics, con el mismo
   patrón `FAILED`×N → `SUCCESS` que describe el apunte.
5. **Entrega**: el hallazgo (credencial válida + falla de control) alimenta
   la fase de Gaining Access del engagement; no incluye explotación
   posterior (eso es Maintaining Access).

## Modos de ejecución (`Modo_Ejecucion`)

Definidos en `plugin.json#x-ethack.executionMode` (contrato estándar del
proyecto, no viene del apunte):

| Modo | Comportamiento esperado |
|---|---|
| `script-kiddie` (default) | Diccionario pequeño y acotado (credenciales de fábrica/comunes), concurrencia baja, para confirmar rápido si la falla del apunte (contraseña por defecto) está presente. |
| `green-hat` | Diccionario más amplio, aplica los límites de `steering/limites-anti-dos.md` de forma explícita y documentada. |
| `pro` | Permite ajustar manualmente diccionario, módulo de Hydra y paralelismo, bajo la responsabilidad del operador de respetar igualmente los límites anti-DoS. |

Valores fuera de `{script-kiddie, green-hat, pro}` se rechazan con
`InvalidModeError` (contrato estándar del modelo de Power).

## Prompts de arranque

- *"Ejecuta fuerza-bruta-login en modo script-kiddie contra `<target>` del
  engagement `<engagement_id>`, usando el diccionario de contraseñas de
  fábrica."*
- *"Valida que `<target>` esté en la allowlist del engagement antes de correr
  Hydra contra el panel de login."*
- *"Interpreta la salida de Hydra y documenta el hallazgo en el formato
  `FAILED`/`SUCCESS` que usa el Blue Team."*
- *"Sube a `green-hat` y repite el ataque con un diccionario más amplio,
  respetando los límites anti-DoS."*

## Artefactos de documentación (`docs/`)

Conforme al contrato `x-ethack.documentation`: `docs/prompt.md`,
`docs/sessions/`, `docs/feedback.md`. La fuente original del apunte vive en
`docs/fuentes/896492b3-4_fuerza_bruta_ejercicio.pdf`.
