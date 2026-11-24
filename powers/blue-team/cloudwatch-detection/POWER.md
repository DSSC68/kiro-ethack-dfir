---
name: "cloudwatch-detection"
displayName: "Blue Team — Detección con CloudWatch Logs"
description: "Power defensivo del Blue Team: detecta la actividad ofensiva del Red Team sobre el laboratorio leyendo, desde Kali y con AWS CLI, los logs del contenedor DVWA centralizados en CloudWatch Logs (/ethack-lab/dvwa). Cubre detección de scanning web, fuerza bruta y uso de webshell mediante filtros y consultas de Logs Insights, con un perfil de solo-lectura (blue-team-ro) que no puede borrar evidencia. Modelada sobre el escenario DVWA del curso Ethical Hacking & Digital Forensics (ULSA)."
keywords: ["blue-team", "defensive", "detection", "cloudwatch", "logs-insights", "aws-cli", "dvwa", "hardening", "scanning"]
author: "ULSA Ethical Hacking & DF"
x-ethack:
  kind: "defensive"
  ecCouncilPhase: null
  forensicsStage: null
  track: "BlueTeam"
  team: "blue-team"
  owner: "Gustavo De la Cruz Tovar"
  tools: ["aws-cli", "cloudwatch-logs", "cloudwatch-logs-insights"]
  allowlistValidation:
    required: true
    source: "engagements/${ENGAGEMENT_ID}/scope.yaml#assets"
    targetParams: ["target", "host", "hostname", "domain", "url"]
  executionMode:
    param: "Modo_Ejecucion"
    allowed: ["script-kiddie", "green-hat", "pro"]
    default: "script-kiddie"
  documentation:
    root: "powers/blue-team/cloudwatch-detection/docs"
    required: ["prompt.md", "sessions/", "feedback.md"]
---

# Blue Team — Detección con CloudWatch Logs

## Overview

Esta Power es la **primera capacidad de detección del Blue Team** del laboratorio.
Convierte el registro del contenedor **DVWA** —que ya viaja a **CloudWatch Logs**
en el log group `/ethack-lab/dvwa`— en **detecciones** de la actividad ofensiva que
el Red Team genera al recorrer el ciclo EC-Council.

No requiere instalar nada en el objetivo ni en Kali: usa la **AWS CLI** (ya
presente en Kali) para leer y consultar los logs en vivo y en frío. Es una Power
de conocimiento (sin servidor MCP): procedimientos, consultas de **CloudWatch Logs
Insights** y reglas de umbral que el agente aplica con la CLI.

> **Ámbito.** Solo laboratorio autorizado. La detección se restringe a los assets
> de la Allowlist_Objetivos (`engagements/dvwa-lab-ulsa/scope.yaml`); el host EC2
> anfitrión no es objetivo salvo `host_is_target: true`.

## Ubicación en el ciclo (contrato x-ethack)

- `kind: defensive`, `track: BlueTeam` — no es una fase EC-Council ni una etapa
  forense; es la capa de **defensa** (Detect / Protect del NIST CSF). Cierra el
  hueco defensivo que la línea base dejaba abierto.
- `team: blue-team` — invocada por `subagente-blue-team`.
- **Herramientas reales:** `aws-cli`, **CloudWatch Logs** (tail/filter) y
  **CloudWatch Logs Insights** (consultas con `stats`/`filter`).

## Credencial de solo-lectura (separación de funciones)

El Blue Team **no** usa el perfil ofensivo de Kali. Usa un perfil separado,
`blue-team` (IAM user `blue-team-ro`), con permisos de **solo lectura** de logs y
**sin** capacidad de borrar o cambiar retención. Esto refuerza el principio de
que la evidencia no se destruye (contrapartida de "covering tracks").

```bash
aws sts get-caller-identity --profile blue-team   # debe ser user/blue-team-ro
```

## Qué detecta (y qué no)

- ✅ **Scanning web** (nikto, gobuster/dirb, nmap con scripts HTTP): ráfagas de
  `404`, rutas inexistentes, user-agents de herramienta.
- ✅ **Fuerza bruta** a `login.php`: muchos `POST` desde una IP.
- ✅ **Webshell**: `POST` al upload seguido de `GET` a `/hackable/uploads/*.php`.
- ❌ **Port scan TCP puro** (`nmap -sS`): no genera HTTP, no aparece en el
  `access.log`. Para la capa de red hace falta VPC Flow Logs o un NIDS (Suricata).
  Esta frontera web↔red es parte de lo que la Power enseña.

## Available Steering Files

Carga la guía según lo que estés detectando:

- **deteccion-scanning-cloudwatch** — Runbook completo: perfil `blue-team`, toma en
  vivo (`aws logs tail`), consultas de Logs Insights para scanning/fuerza
  bruta/webshell, regla de umbral y metric filter + alarma de CloudWatch.

Para leerlo: acción `readSteering` con `powerName="cloudwatch-detection"` y
`steeringFile="deteccion-scanning-cloudwatch"`.

## Modos de ejecución (Modo_Ejecucion)

- **script-kiddie** (default) — ejecuta las consultas tal cual, reporta la IP
  atacante y la fase detectada.
- **green-hat** — además explica *por qué* cada patrón (qué significa una ráfaga de
  404, por qué los POST a `login.php` indican fuerza bruta) y el parseo del log.
- **pro** — permite afinar reglas/umbral y crear metric filters + alarmas propias
  acotadas al lab.

## Onboarding

### Prerrequisitos
- Perfil `blue-team` configurado en Kali (`aws configure --profile blue-team`),
  región `us-east-1`.
- Log group `/ethack-lab/dvwa` activo (lo crea la plantilla del lab, opción A).

### Primer uso
```bash
aws logs tail /ethack-lab/dvwa --profile blue-team --region us-east-1 --follow
```

## Entregable

Un **hallazgo de detección** con su evidencia: IP atacante, fase EC-Council
detectada, la línea de log / consulta que lo sustenta y la hora (UTC). Escala al
`subagente-digital-forensics` si requiere recolección formal de evidencia.
