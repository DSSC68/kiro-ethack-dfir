# prompt.md — cloudwatch-detection (Blue Team)

Artefacto de documentación (Req 15.1). Prompt base con el que el
`subagente-blue-team` invoca esta Power.

## Intención

Detectar la actividad ofensiva del Red Team sobre el laboratorio DVWA leyendo el
log del contenedor en CloudWatch (`/ethack-lab/dvwa`), desde Kali y con la AWS CLI
(perfil de solo-lectura `blue-team`).

## Prompt de referencia

> Eres el Blue Team. Usando el perfil `blue-team` y la AWS CLI, revisa el log
> group `/ethack-lab/dvwa` de los últimos N minutos y determina si hay:
> (1) scanning web (ráfaga de 404 / user-agents de escáner),
> (2) fuerza bruta a `login.php`,
> (3) uso de webshell en `/hackable/uploads/`.
> Para cada detección, reporta la IP atacante, la fase EC-Council, la evidencia
> (línea de log o consulta) y la hora UTC. Valida que el objetivo observado esté
> en la Allowlist_Objetivos (`scope.yaml`). Modo por defecto: script-kiddie.

## Entradas

- `Modo_Ejecucion` (script-kiddie | green-hat | pro), default script-kiddie.
- Ventana de tiempo a analizar.

## Salida esperada

Hallazgo(s) de detección con evidencia de sustento y línea de tiempo.
