# Detección de scanning desde Kali con AWS CLI (CloudWatch Logs)

Guía operativa del Blue Team para detectar la actividad ofensiva del Red Team
leyendo el log del contenedor DVWA en CloudWatch (`/ethack-lab/dvwa`), usando la
AWS CLI desde Kali con el perfil de solo-lectura `blue-team`.

## 0. Preparación

```bash
# Confirma que usas la credencial de solo-lectura (no la ofensiva)
aws sts get-caller-identity --profile blue-team     # -> user/blue-team-ro
```

Variables de trabajo:

```bash
LG=/ethack-lab/dvwa
PROF=blue-team
REG=us-east-1
```

## Formato del log y patrón de parseo

Las líneas de acceso de Apache (combined log format):

```
10.50.0.68 - - [20/Nov/2026:00:04:10 +0000] "GET /login.php HTTP/1.1" 200 2365 "-" "curl/8.18.0"
```

Patrón base para **Logs Insights** (reutilizable en casi todas las consultas):

```
fields @timestamp, @message
| parse @message '* - - [*] "* * *" * * "*" "*"' as ip, ts, method, url, proto, status, bytes, referer, ua
```

## 1. Toma en vivo

Tap simple a los 404 (señal de barrido) mientras el Red Team ataca:

```bash
aws logs tail $LG --profile $PROF --region $REG --follow --filter-pattern '"404"'
```

Todo el tráfico en vivo (sin filtro):

```bash
aws logs tail $LG --profile $PROF --region $REG --follow
```

## 2. Detección de scanning (404 por IP + herramienta)

```bash
QID=$(aws logs start-query --profile $PROF --region $REG \
  --log-group-name $LG \
  --start-time $(date -d '-30 min' +%s) --end-time $(date +%s) \
  --query-string 'fields @timestamp,@message
    | parse @message "* - - [*] \"* * *\" * * \"*\" \"*\"" as ip,ts,method,url,proto,status,bytes,referer,ua
    | filter status="404"
    | stats count(*) as hits by ip, ua
    | sort hits desc' \
  --query 'queryId' --output text)
sleep 5
aws logs get-query-results --profile $PROF --region $REG --query-id $QID
```

Detección por user-agent de escáner (nikto/gobuster/nmap/curl):

```
| parse @message '* - - [*] "* * *" * * "*" "*"' as ip,ts,method,url,proto,status,bytes,referer,ua
| filter ua like /(?i)(nikto|gobuster|dirb|nmap|masscan|sqlmap|python-requests|curl)/
| stats count(*) as hits by ip, ua | sort hits desc
```

**Regla de umbral:** más de **N** `404` (p. ej. 20) desde una misma IP en pocos
segundos ⇒ **scanning**. Registra IP, herramienta (ua), hora de inicio/fin.

## 3. Fuerza bruta de login

```
| parse @message '* - - [*] "* * *" * * "*" "*"' as ip,ts,method,url,proto,status,bytes,referer,ua
| filter url like "/login.php" and method = "POST"
| stats count(*) as intentos by ip | sort intentos desc
```

Señal de **login exitoso tras fuerza bruta**: ráfaga de `302` (redirección post
login) después de varios intentos desde la misma IP.

## 4. Uso de webshell (maintaining access)

```
| parse @message '* - - [*] "* * *" * * "*" "*"' as ip,ts,method,url,proto,status,bytes,referer,ua
| filter url like "/hackable/uploads/" and url like ".php"
| sort @timestamp asc
```

Registra el nombre del archivo subido y cada acceso posterior (eso es el atacante
"manteniendo acceso").

## 5. Detección automática: metric filter + alarma

Para que CloudWatch avise solo (lo puede crear el docente/Líder Técnico con un
perfil con permisos de escritura; el perfil `blue-team` es solo lectura):

```bash
# Métrica: cuenta de respuestas 404 en el log group
aws logs put-metric-filter --region $REG \
  --log-group-name $LG \
  --filter-name dvwa-404 \
  --filter-pattern '"404"' \
  --metric-transformations metricName=Dvwa404,metricNamespace=EthackBlueTeam,metricValue=1

# Alarma: >20 cuatrocientos-cuatro en 1 minuto -> posible scanning
aws cloudwatch put-metric-alarm --region $REG \
  --alarm-name dvwa-posible-scanning \
  --namespace EthackBlueTeam --metric-name Dvwa404 \
  --statistic Sum --period 60 --threshold 20 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 1 \
  --treat-missing-data notBreaching
```

## Frontera web ↔ red (importante)

Esto detecta scanning **web** (HTTP). Un `nmap -sS` de puertos TCP **no** aparece
aquí porque no genera peticiones HTTP; para esa capa se necesitan **VPC Flow Logs**
o un **NIDS** (Suricata). Explícalo al reportar: "cubro la capa de aplicación".

## Qué NO puede hacer este perfil

El perfil `blue-team` (IAM `blue-team-ro`) **no** puede `DeleteLogGroup` ni
`PutRetentionPolicy`: aunque el atacante borre los logs dentro del contenedor, la
copia en CloudWatch permanece y el Blue Team no puede destruirla por error. Esa es
la contrapartida de la fase *Covering Tracks*.
