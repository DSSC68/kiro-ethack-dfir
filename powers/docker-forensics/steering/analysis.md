# Análisis offline del paquete adquirido

Una vez que el paquete está resguardado en el bastión, el análisis se hace **offline** sobre los artefactos, sin volver a tocar el entorno vivo. Responde la pregunta operativa: *¿qué hicieron realmente?*

---

## 0. Preparar el área de trabajo

```bash
cd ~/forensics/container-forensics-<TS>
ls -R
# Revalidar integridad antes de analizar
sha256sum -c MANIFEST.sha256 | grep -v ': OK$' || echo "Todos los artefactos íntegros"
```

Trabaja sobre copias; no modifiques los originales.

## 1. Expandir el rootfs (imagen forense del contenedor)

```bash
mkdir -p rootfs && tar xzf fs/rootfs.tar.gz -C rootfs
# El webroot vivo del contenedor queda en rootfs/var/www/html
ls -la rootfs/var/www/html
```

Qué buscar en el rootfs:
- Archivos nuevos o modificados en el webroot que no pertenezcan a la app base.
- Binarios o scripts en rutas inusuales (`/tmp`, `/dev/shm`, `/var/www/html/...`).
- Entradas de persistencia (cron, `.bashrc`, claves SSH añadidas) si el escenario lo justifica.

## 2. Timeline por mtime

El timeline capturado (`timeline/filesystem-timeline.psv`) tiene formato `mtime|atime|ctime|modo|uid|gid|size|ruta`.

```bash
# Top 50 por modificación más reciente, con fecha legible
sort -t'|' -k1 -rn timeline/filesystem-timeline.psv | head -50 | \
  awk -F'|' '{ cmd="date -u -d @"int($1)" +%Y-%m-%dT%H:%M:%SZ"; cmd | getline d; close(cmd); print d, $4, $8, $NF }'

# Acotar a una ventana del incidente (ejemplo: epoch entre T0 y T1)
awk -F'|' '$1>=1731900000 && $1<=1731910000 {print}' timeline/filesystem-timeline.psv
```

Para un timeline formal, puedes convertir a bodyfile y usar `mactime` (Sleuth Kit) sobre el rootfs expandido.

## 3. Revisar la base de datos

```bash
# Importar en una BD desechable local
mysql -uroot -e "CREATE DATABASE dvwa_ev;" 2>/dev/null
mysql -uroot dvwa_ev < db/dvwa.sql

# Usuarios: ¿intactos (hashes MD5 por defecto) o alterados?
mysql -uroot dvwa_ev -e "SELECT user_id,user,password FROM users;"
# Guestbook: ¿XSS almacenado / inyección persistida?
mysql -uroot dvwa_ev -e "SELECT * FROM guestbook;"
```

Compara los usuarios contra los 5 por defecto de DVWA (`admin`, `gordonb`, `1337`, `pablo`, `smithy`) y sus hashes MD5 de fábrica. Cualquier hash distinto, usuario nuevo o fila de guestbook con `<script>`/payload es hallazgo.

## 4. Interpretar `docker diff`

```bash
cat container/diff.txt | sort
```

- `C`/`A` en `/var/lib/php/sessions/` → navegación normal (sesiones PHP). Benigno.
- `A` en el webroot (`/var/www/html/...`) o en `hackable/uploads/` → posible webshell o archivo subido: investigar.
- Cambios en binarios del sistema → sospechoso.

Correlaciona cada `A`/`C` sospechoso con el timeline y los logs.

## 5. Buscar webshells y uploads

```bash
tar tzf uploads/uploads.tar.gz          # ¿qué se subió?
cat uploads/listing.txt
# Buscar patrones de webshell dentro del rootfs expandido
grep -rEn "eval\(|base64_decode\(|system\(|passthru\(|shell_exec\(" rootfs/var/www/html 2>/dev/null | head
```

## 6. Logs de Apache: autenticación, ataques y escaneos

```bash
# IPs externas y su volumen
awk '{print $1}' logs/apache-access.log | grep -vE '^(127\.|10\.|172\.|192\.168\.)' | sort | uniq -c | sort -rn

# ¿Hubo login? (autenticación efectiva)
grep "POST /login.php" logs/apache-access.log

# Ataques a módulos vulnerables
grep "/vulnerabilities/" logs/apache-access.log

# Huella de escaneo automatizado (nmap http-* NSE)
grep -E "nmaplowercheck|/sdk|/evox/about|/HNAP1" logs/apache-access.log
```

Interpretación típica:
- Solo `GET /` y `GET /login.php` desde navegador + ráfaga de rutas 404 de nmap → **reconocimiento y escaneo**, sin explotación.
- `POST /login.php` seguido de peticiones a `/vulnerabilities/...` → autenticaron y ejercitaron módulos.
- `error.log` con solo arranque de Apache y avisos benignos → sin trazas de ataque.

## 7. Veredicto

Cruza las seis fuentes (diff, timeline, BD, uploads, access.log, error.log) y concluye en términos de **qué fases EC-Council** alcanzaron (recon / scanning / gaining access / maintaining). Enlaza cada afirmación a la línea de evidencia concreta del manifiesto. La plantilla del reporte está en el steering **chain-of-custody**.
