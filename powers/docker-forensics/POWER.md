---
name: "docker-forensics"
displayName: "Docker Container Forensics"
description: "Guía de análisis forense en caliente de contenedores Docker en un laboratorio autorizado: adquisición de evidencia a través de un bastión SSH, resguardo verificado por hash, análisis offline del rootfs/BD/logs y cadena de custodia. Modelada sobre el escenario DVWA del curso Ethical Hacking & Digital Forensics (ULSA)."
keywords: ["docker-forensics", "container-forensics", "dfir", "live-response", "chain-of-custody", "dvwa"]
author: "ULSA Ethical Hacking & DF"
x-ethack:
  kind: "forensics"
  ecCouncilPhase: null
  forensicsStage: "Collect"
  team: "digital-forensics"
  owner: "Gustavo De la Cruz Tovar"
  tools: ["docker", "ssh", "sha256sum", "mactime", "aws-cli"]
  allowlistValidation:
    required: true
    source: "engagements/${ENGAGEMENT_ID}/scope.yaml#assets"
    targetParams: ["target", "host", "hostname", "domain", "url"]
  executionMode:
    param: "Modo_Ejecucion"
    allowed: ["script-kiddie", "green-hat", "pro"]
    default: "script-kiddie"
  documentation:
    root: "powers/docker-forensics/docs"
    required: ["prompt.md", "sessions/", "feedback.md"]
---

# Docker Container Forensics

## Overview

Esta Power documenta cómo hacer análisis forense **en caliente** (live response) de un contenedor Docker en ejecución dentro de un laboratorio autorizado, y cómo preservar la evidencia de forma defendible. Nace del escenario real del curso: un contenedor **DVWA** (`vulnerables/web-dvwa`) corriendo sobre un host EC2 al que solo se llega a través de una instancia **Kali** que actúa de bastión (salto SSH / `ProxyCommand`).

El flujo cubre las cuatro fases clásicas de DFIR aplicadas a un contenedor:

1. **Preparación y acceso** — establecer el salto SSH al host del contenedor sin tratar al host como objetivo.
2. **Adquisición** — recolectar metadatos volátiles, el rootfs completo (`docker export`), la base de datos, logs, sesiones, uploads y un timeline del filesystem; empaquetar y calcular hashes.
3. **Resguardo** — transferir el paquete a un almacén (la instancia Kali / un snapshot EBS) verificando integridad extremo a extremo.
4. **Análisis y reporte** — trabajar offline sobre el paquete adquirido (montar el rootfs, correr un timeline tipo mactime, revisar el dump de BD y los logs) y documentar hallazgos con cadena de custodia.

Es una Power de conocimiento (sin servidor MCP): son procedimientos, scripts y plantillas que el agente aplica con las herramientas normales de shell y AWS CLI.

> **Ámbito de uso.** Pensada exclusivamente para laboratorios y entornos **autorizados** (como el del curso). La adquisición toca datos y estado de un sistema en ejecución; hazlo solo donde tengas permiso explícito.

## Available Steering Files

Carga el archivo de la fase en la que estés trabajando:

- **acquisition** — Procedimiento y script completo de adquisición en caliente del contenedor (metadatos, `docker export`, dump de BD, logs, timeline, manifiesto SHA256) y transferencia verificada al bastión.
- **analysis** — Análisis offline sobre el paquete adquirido: montar el rootfs, correr un timeline por mtime, revisar el dump de BD, interpretar `docker diff` y los logs de Apache, buscar webshells/persistencia.
- **chain-of-custody** — Reglas de preservación de evidencia, plantilla de cadena de custodia, snapshots EBS como respaldo inmutable y estructura del reporte forense.

Para leerlos: usa la acción `readSteering` con `powerName="docker-forensics"` y el `steeringFile` correspondiente.

## Onboarding

### Prerequisitos

- Acceso autorizado al laboratorio y a la llave SSH (`.pem`) del entorno.
- `docker` instalado en el host que ejecuta el contenedor (acceso `sudo`).
- `aws` CLI configurado con el perfil del laboratorio (para snapshots EBS).
- Conectividad al contenedor **solo a través del bastión** (no exponer el host como objetivo).
- Herramientas en el analista: `tar`, `gzip`, `sha256sum`, `mysql`/`mariadb` client (para revisar el dump), y opcionalmente `mactime`/Sleuth Kit.

### Topología del escenario (referencia)

```
Analista ──SSH(.pem)──▶ Kali (bastión / público)
                           │  ProxyCommand -W %h:%p
                           ▼
                        Host EC2 del contenedor (privado)  ──docker──▶ contenedor DVWA
```

- La IP de egreso del analista **rota**: reautoriza tu IP en el Security Group del bastión (puerto 22) antes de cada operación con `curl -s https://checkip.amazonaws.com`.
- El contenedor nunca se expone a Internet: su puerto 80 queda restringido por Security Group / prefix list.

### Establecer el salto SSH

```bash
KEY=/ruta/a/llave.pem
BASTION_USER=kali            # usuario del bastión
BASTION_IP=<IP_PUBLICA_KALI>
HOST_USER=ec2-user           # usuario del host del contenedor
HOST_PRIV_IP=<IP_PRIVADA_HOST_DVWA>

# ProxyCommand que tuneliza a través del bastión
PX="ssh -i $KEY -o StrictHostKeyChecking=no -o ConnectTimeout=12 -W %h:%p ${BASTION_USER}@${BASTION_IP}"

# Ejecutar un comando en el host del contenedor
ssh -i "$KEY" -o StrictHostKeyChecking=no -o ConnectTimeout=12 \
  -o ProxyCommand="$PX" ${HOST_USER}@${HOST_PRIV_IP} "sudo docker ps"
```

> **Nota sobre comillas.** El doble salto con comillas anidadas rompe fácil el parser de la shell. Prefiere **dejar los scripts en archivos** y copiarlos con `scp` (vía el mismo `ProxyCommand`) en vez de incrustar heredocs/comillas dentro de comandos SSH.

### Verificación rápida

```bash
# ¿El contenedor está vivo y cuánto escribió sobre la capa de imagen?
ssh ... ${HOST_USER}@${HOST_PRIV_IP} "sudo docker ps -s --filter name=<contenedor>"
# Espacio en disco ANTES de exportar (ojo: /tmp suele ser tmpfs en RAM)
ssh ... ${HOST_USER}@${HOST_PRIV_IP} "df -h / /var/tmp"
```

## Common Workflows

### Workflow 1: Adquisición en caliente del contenedor

Objetivo: capturar todo lo necesario para DF sin apagar el contenedor, y empaquetarlo con manifiesto de hashes.

Resumen (detalle completo y script en el steering **acquisition**):

1. Elegir un directorio de trabajo **en disco real** del host (`/var/tmp`, no `/tmp` si es tmpfs).
2. Recolectar metadatos volátiles: `docker inspect`, `docker diff`, `docker logs`, `docker top`, procesos y conexiones *dentro* del contenedor.
3. Exportar el rootfs: `docker export <contenedor> | gzip > rootfs.tar.gz`.
4. Dump de la BD de la app (`mysqldump`), copia de logs de Apache, sesiones PHP, uploads y webroot.
5. Generar un timeline del filesystem (`find / -printf` con mtime/atime/ctime).
6. Calcular `MANIFEST.sha256` de todos los artefactos y empaquetar en un `.tar.gz` con su hash.

### Workflow 2: Resguardo verificado en el bastión

Objetivo: dejar la evidencia **en la instancia Kali** (y en un snapshot EBS) con integridad comprobada.

1. Transferir el paquete por la **red interna de la VPC** (host → bastión) para no pasar GB por la WAN.
2. Verificar el SHA256 del paquete contra el generado en origen.
3. Desempaquetar y validar el `MANIFEST.sha256` interno.
4. Tomar un **snapshot EBS** del volumen del bastión: congela la evidencia de forma inmutable y barata.

Detalle en el steering **acquisition** (transferencia) y **chain-of-custody** (snapshots).

### Workflow 3: Análisis offline y reporte

Objetivo: responder "¿qué hicieron?" sin tocar más el entorno vivo.

1. Expandir `rootfs.tar.gz` en una carpeta de solo lectura.
2. Correr un timeline por mtime y acotar la ventana del incidente.
3. Importar `dvwa.sql` en una BD desechable y comparar contra el estado por defecto (usuarios, guestbook).
4. Interpretar `docker diff` (¿archivos nuevos en el webroot? ¿webshells en uploads?).
5. Revisar `access.log`/`error.log` buscando autenticación, ataques a módulos y escaneos.
6. Redactar el reporte con veredicto y evidencia enlazada al manifiesto.

Detalle en el steering **analysis** y la plantilla en **chain-of-custody**.

## Best Practices

- **Orden de volatilidad.** Captura primero lo volátil (procesos, conexiones, `docker diff`) y luego lo estable (rootfs, BD, logs).
- **No contamines la escena.** Escribe la evidencia fuera del contenedor; evita instalar paquetes dentro del contenedor durante la adquisición.
- **Hashea todo.** Un `MANIFEST.sha256` por adquisición y un hash del paquete final; verifica tras cada transferencia.
- **Integridad de la transferencia.** Compara el hash en destino contra el de origen; registra el resultado.
- **Snapshot como respaldo inmutable.** Un snapshot EBS del volumen donde vive la evidencia es tu copia maestra entre clases.
- **Cadena de custodia.** Registra operador, fecha UTC, método y ubicación en un archivo `00-ACQUISITION-INFO.txt` dentro del paquete.
- **Mínimo privilegio y limpieza.** Si copias la llave `.pem` al bastión para una transferencia interna, bórrala con `shred -u` al terminar.
- **Dos puntos en el tiempo.** Si adquieres después de haber analizado en caliente, anótalo: la adquisición es un segundo snapshot temporal, no el mismo instante.

## Troubleshooting

### `docker export` llena el disco o falla
**Causa:** se exportó a `/tmp`, que suele ser `tmpfs` (RAM) y es pequeño.
**Solución:** usa `/var/tmp` u otra ruta en el disco real; verifica `df -h` antes. El rootfs comprimido de DVWA ronda ~150 MB, pero reserva holgura.

### SSH al host del contenedor falla con timeout
**Causa:** tu IP de egreso cambió y el Security Group del bastión ya no la permite.
**Solución:** `MYIP=$(curl -s https://checkip.amazonaws.com)` y reautoriza 22/tcp para `$MYIP/32` en el SG del bastión.

### El parser de la shell rompe con comillas/heredocs en el doble salto
**Causa:** comillas anidadas entre tu shell, el bastión y el host.
**Solución:** escribe el script en un archivo local y cópialo con `scp -o ProxyCommand=...`; ejecútalo con `bash /ruta/script.sh`.

### `sha256sum -c` falla tras transferir
**Causa:** el archivo `.sha256` lleva la ruta absoluta del origen (`/var/tmp/...`) que no existe en destino.
**Solución:** compara solo el hash (`sha256sum archivo | awk '{print $1}'`) contra el valor esperado, o regenera el `.sha256` con rutas relativas.

### El script de AWS da `NoCredentials` pese a tener perfil válido
**Causa:** el script hace `export AWS_PROFILE` sin asignarle valor (exporta vacío).
**Solución:** pásalo en el entorno (`AWS_PROFILE=<perfil> ./script.sh`) o corrige a `export AWS_PROFILE="$PROFILE"`.

---

**Tipo:** Knowledge Base Power (sin mcp.json)
**Escenario base:** contenedor DVWA sobre EC2, bastión Kali, curso Ethical Hacking & DF (ULSA)
