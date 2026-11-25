# Realimentación del resultado — `fuerza-bruta-login`

> Artefacto_Documentacion obligatorio. Registra la realimentación del resultado
> del Power al chat/contexto de Kiro: qué salió bien, qué corregiste y qué
> aprendiste al probarlo contra el Entorno_Laboratorio.

## Qué funcionó

- El `plugin.json` quedó completo y es JSON válido: `name`, `author`,
  `owner`, `ecCouncilPhase: "Gaining Access"` y `tools: [Hydra]`, con
  `allowlistValidation` y `executionMode` conservados sin tocar, tal como se
  pidió.
- El `README.md` y las 4 guías de `steering/` (`preparacion-diccionario.md`,
  `ejecucion-hydra-formulario.md`, `interpretacion-resultados.md`,
  `limites-anti-dos.md`) separan con claridad lo que viene del apunte de lo
  agregado, lo que hace auditable qué parte del Power está respaldada por la
  fuente y cuál no.
- El Power quedó coherente con el rol de Red Team: valida el objetivo contra
  `scope.yaml#assets` antes de ejecutar, y documenta el hallazgo en el
  formato que puede consumir Blue Team / Digital Forensics
  (`producesActionLog: true`).

## Qué se corrigió tras la realimentación

- **El apunte resultó ser forense, no operativo.** Al extraer el PDF
  (`docs/fuentes/4_fuerza_bruta_ejercicio.pdf`), el contenido real era un
  ejercicio de lectura de `tcpdump` sobre un incidente ya ocurrido (cómo se
  ve un sitio comprometido, cómo distinguirlo de un DNS caído o un SYN
  flood, cómo prevenir la fuerza bruta) — **no un tutorial de Hydra**. No
  contenía ningún comando de ejecución de fuerza bruta.
- Por esa razón, **todos los comandos de Hydra en las guías de `steering/`
  quedaron marcados explícitamente como `(añadido)`**, en vez de
  presentarlos como si vinieran del apunte. Esto se aplicó en
  `ejecucion-hydra-formulario.md` (sintaxis `http-post-form`, SSH),
  `preparacion-diccionario.md` (armado de listas de usuarios/contraseñas),
  `limites-anti-dos.md` (parámetros `-t`/`-w`) e
  `interpretacion-resultados.md` (mapeo de la salida real de Hydra al
  patrón de log que sí describe el apunte).
- **Se mantuvo el Power como ofensivo** (`kind: "offensive"`,
  `ecCouncilPhase: "Gaining Access"`) en lugar de reconvertirlo a
  defensivo, porque el objetivo pedido era un Power de Red Team que
  colabora con Blue Team entregando un hallazgo reproducible — ese
  hallazgo es justamente el insumo que después usa el Power hermano
  `deteccion-fuerza-bruta` desde el lado de detección.

## Pendientes / mejoras futuras

- **Validar los comandos de Hydra contra un servicio vivo.** No se contó
  con acceso a un objetivo de laboratorio real durante esta sesión; los
  comandos `(añadido)` de `ejecucion-hydra-formulario.md` son sintaxis
  estándar de Hydra pero no se ejecutaron ni se confirmó la cadena de
  éxito/fallo (`F=`/`S=`) contra un formulario real. Antes de usarlos en el
  engagement, confirmar la cadena de respuesta real del servicio objetivo.
- Evaluar si conviene enlazar explícitamente este Power con
  `deteccion-fuerza-bruta` en la documentación (p. ej. referenciar que el
  hallazgo de uno alimenta al otro) una vez que el Catalogador soporte
  relaciones entre Powers.
