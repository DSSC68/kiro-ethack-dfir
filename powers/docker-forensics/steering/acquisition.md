# Adquisición en caliente del contenedor + resguardo verificado

Procedimiento completo y script reutilizable para capturar toda la evidencia de un contenedor Docker en ejecución y dejarla, verificada por hash, en el bastión.

---

## 1. Variables del entorno

```bash
KEY=/ruta/a/llave.pem
BASTION_USER=kali
BASTION_IP=<IP_PUBLICA_KALI>
HOST_USER=ec2-user
HOST_PRIV_IP=<IP_PRIVADA_HOST_DVWA>
CONTAINER=dvwa
IMAGE=vulnerables/web-dvwa
PX="ssh -i $KEY -o StrictHostKeyChecking=no -o ConnectTimeout=12 -W %h:%p ${BASTION_USER}@${BASTION_IP}"

# Atajo para ejecutar en el host del contenedor
onhost() { ssh -i "$KEY" -o StrictHostKeyChecking=no -o ConnectTimeout=12 -o ProxyCommand="$PX" ${HOST_USER}@${HOST_PRIV_IP} "$@"; }
```

Antes de empezar, reautoriza tu IP de egreso en el SG del bastión (puerto 22) y comprueba disco:

```bash
MYIP=$(curl -s https://checkip.amazonaws.com)
# aws ec2 authorize-security-group-ingress --group-id <SG_BASTION> \
#   --ip-permissions IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges="[{CidrIp=$MYIP/32,Description=forense}]" ...
onhost "df -h / /var/tmp; sudo docker ps -s --filter name=$CONTAINER"
```

> `/tmp` suele ser `tmpfs` (RAM). Usa `/var/tmp` (disco real) para la adquisición.

## 2. Script de adquisición

Guarda esto como `acq_container.sh`, cópialo al host y ejecútalo. Está parametrizado por contenedor e imagen.

```bash
#!/bin/bash
# Adquisicion forense en caliente de un contenedor Docker.
set -u
C="${1:-dvwa}"                 # nombre del contenedor
IMG="${2:-vulnerables/web-dvwa}"
TS=$(date -u +%Y%m%dT%H%M%SZ)
BASE="/var/tmp/container-forensics-${TS}"
mkdir -p "$BASE"/{host,container,image,webroot,logs,sessions,uploads,db,timeline,fs}
echo "[*] Carpeta: $BASE"

# 0. Cadena de custodia
cat > "$BASE/00-ACQUISITION-INFO.txt" <<EOF
Adquisicion forense en caliente - Contenedor $C
Fecha (UTC)  : $(date -u +%Y-%m-%dT%H:%M:%SZ)
Host         : $(hostname) / $(hostname -I 2>/dev/null)
Operador     : $(whoami) (via bastion SSH)
Imagen       : $IMG
Metodo       : docker export + mysqldump + copia de logs/sesiones/uploads + timeline
Nota         : Laboratorio autorizado. Contenedor en ejecucion (live response).
EOF

# 1. Host
date -u > "$BASE/host/date-utc.txt"
uname -a > "$BASE/host/uname.txt" 2>&1
sudo docker version > "$BASE/host/docker-version.txt" 2>&1
sudo docker ps -a -s > "$BASE/host/docker-ps.txt" 2>&1

# 2. Contenedor (volatil primero)
sudo docker inspect "$C"  > "$BASE/container/inspect.json" 2>&1
sudo docker diff "$C"     > "$BASE/container/diff.txt" 2>&1
sudo docker logs "$C"     > "$BASE/container/logs.txt" 2>&1
sudo docker top "$C" aux  > "$BASE/container/top.txt" 2>&1
sudo docker port "$C"     > "$BASE/container/port.txt" 2>&1
sudo docker stats --no-stream "$C" > "$BASE/container/stats.txt" 2>&1
sudo docker exec "$C" sh -c 'ps aux 2>/dev/null || ps -ef' > "$BASE/container/ps-inside.txt" 2>&1
sudo docker exec "$C" sh -c 'netstat -tunap 2>/dev/null || ss -tunap 2>/dev/null' > "$BASE/container/netstat-inside.txt" 2>&1
sudo docker exec "$C" sh -c 'cat /etc/hosts; echo ---; cat /etc/passwd' > "$BASE/container/etc-inside.txt" 2>&1

# 3. Imagen
sudo docker history --no-trunc "$IMG" > "$BASE/image/history.txt" 2>&1
sudo docker image inspect "$IMG"       > "$BASE/image/inspect.json" 2>&1

# 4. Webroot + config
sudo docker exec "$C" sh -c 'cat /var/www/html/config/config.inc.php' > "$BASE/webroot/config.inc.php" 2>&1
sudo docker exec "$C" sh -c 'cd /var/www/html && tar czf - .' > "$BASE/webroot/html.tar.gz" 2>/dev/null

# 5. Logs Apache
sudo docker exec "$C" sh -c 'cat /var/log/apache2/access.log 2>/dev/null' > "$BASE/logs/apache-access.log" 2>&1
sudo docker exec "$C" sh -c 'cat /var/log/apache2/error.log 2>/dev/null'  > "$BASE/logs/apache-error.log" 2>&1

# 6. Sesiones PHP
sudo docker exec "$C" sh -c 'cd /var/lib/php/sessions 2>/dev/null && tar czf - . 2>/dev/null' > "$BASE/sessions/php-sessions.tar.gz" 2>/dev/null
sudo docker exec "$C" sh -c 'ls -la /var/lib/php/sessions 2>/dev/null' > "$BASE/sessions/listing.txt" 2>&1

# 7. Uploads
sudo docker exec "$C" sh -c 'ls -laR /var/www/html/hackable/uploads 2>/dev/null' > "$BASE/uploads/listing.txt" 2>&1
sudo docker exec "$C" sh -c 'cd /var/www/html/hackable/uploads 2>/dev/null && tar czf - . 2>/dev/null' > "$BASE/uploads/uploads.tar.gz" 2>/dev/null

# 8. Base de datos (prueba creds comunes del image DVWA)
for CRED in "root:p@ssw0rd" "root:" "root:vulnerables"; do
  U="${CRED%%:*}"; P="${CRED#*:}"; [ -n "$P" ] && PARG="-p$P" || PARG=""
  if sudo docker exec "$C" sh -c "mysqldump -u$U $PARG --databases dvwa 2>/dev/null" > "$BASE/db/dvwa.sql" 2>/dev/null && [ -s "$BASE/db/dvwa.sql" ]; then
    echo "dump OK usuario=$U" > "$BASE/db/creds-detected.txt"
    sudo docker exec "$C" sh -c "mysql -u$U $PARG -e 'SHOW DATABASES; SELECT user,host FROM mysql.user;' 2>/dev/null" > "$BASE/db/mysql-overview.txt" 2>&1
    break
  fi
done
[ -s "$BASE/db/dvwa.sql" ] || echo "NO se pudo generar dump" > "$BASE/db/creds-detected.txt"

# 9. Timeline del filesystem del contenedor
sudo docker exec "$C" sh -c "find / -xdev -printf '%T@|%A@|%C@|%m|%u|%g|%s|%p\n' 2>/dev/null" > "$BASE/timeline/filesystem-timeline.psv" 2>/dev/null
sort -t'|' -k1 -rn "$BASE/timeline/filesystem-timeline.psv" 2>/dev/null | head -200 > "$BASE/timeline/recent-200-by-mtime.txt"

# 10. Export del rootfs (imagen forense del contenedor)
echo "[*] Exportando rootfs..."
sudo docker export "$C" | gzip > "$BASE/fs/rootfs.tar.gz" 2>/dev/null

# 11. Manifiesto de hashes
( cd "$BASE" && find . -type f ! -name MANIFEST.sha256 -print0 | sort -z | xargs -0 sha256sum > MANIFEST.sha256 )

# 12. Empaquetar
TARBALL="/var/tmp/container-forensics-${TS}.tar.gz"
( cd /var/tmp && tar czf "$TARBALL" "container-forensics-${TS}" )
sha256sum "$TARBALL" > "${TARBALL}.sha256"
echo "[DONE] $TARBALL"; du -sh "$BASE" "$TARBALL"; cat "${TARBALL}.sha256"
```

Ejecutar:

```bash
scp -i "$KEY" -o StrictHostKeyChecking=no -o ProxyCommand="$PX" acq_container.sh ${HOST_USER}@${HOST_PRIV_IP}:/var/tmp/acq_container.sh
onhost "bash /var/tmp/acq_container.sh $CONTAINER $IMAGE"
```

## 3. Verificar lo capturado antes de transferir

```bash
D=/var/tmp/container-forensics-<TS>
onhost "du -ah $D | sort -k2; echo '--- db ---'; cat $D/db/creds-detected.txt; head -3 $D/db/dvwa.sql; wc -l $D/MANIFEST.sha256"
```

Confirma que existan: `fs/rootfs.tar.gz` (no vacío), `db/dvwa.sql` (con contenido), `MANIFEST.sha256` con todas las líneas.

## 4. Resguardo en el bastión (transferencia por la VPC)

Para no mover GB por la WAN, transfiere **host → bastión** por la red interna. El bastión necesita la llave para hablar con el host; cópiala temporal y bórrala con `shred`.

```bash
TB=container-forensics-<TS>.tar.gz
# Preparar destino y subir la llave al bastion (permisos estrictos)
ssh -i "$KEY" -o StrictHostKeyChecking=no ${BASTION_USER}@${BASTION_IP} "mkdir -p ~/forensics && chmod 700 ~/forensics"
scp -i "$KEY" -o StrictHostKeyChecking=no "$KEY" ${BASTION_USER}@${BASTION_IP}:/home/${BASTION_USER}/.labkey.pem
ssh -i "$KEY" -o StrictHostKeyChecking=no ${BASTION_USER}@${BASTION_IP} "chmod 600 ~/.labkey.pem"

# Desde el bastion, jalar el paquete del host por la VPC
ssh -i "$KEY" -o StrictHostKeyChecking=no ${BASTION_USER}@${BASTION_IP} bash -s <<REMOTE
set -e
cd ~/forensics
scp -i ~/.labkey.pem -o StrictHostKeyChecking=no ${HOST_USER}@${HOST_PRIV_IP}:/var/tmp/${TB} .
scp -i ~/.labkey.pem -o StrictHostKeyChecking=no ${HOST_USER}@${HOST_PRIV_IP}:/var/tmp/${TB}.sha256 .
REMOTE

# Verificar integridad por hash (el .sha256 trae ruta absoluta de origen: compara solo el hash)
EXPECTED=$(onhost "awk '{print \$1}' /var/tmp/${TB}.sha256")
ACTUAL=$(ssh -i "$KEY" -o StrictHostKeyChecking=no ${BASTION_USER}@${BASTION_IP} "sha256sum ~/forensics/${TB} | awk '{print \$1}'")
[ "$EXPECTED" = "$ACTUAL" ] && echo "INTEGRIDAD OK" || echo "MISMATCH"

# Desempaquetar y validar manifiesto interno
ssh -i "$KEY" -o StrictHostKeyChecking=no ${BASTION_USER}@${BASTION_IP} \
  "cd ~/forensics && tar xzf ${TB} && cd container-forensics-<TS> && sha256sum -c MANIFEST.sha256 | tail -5"

# Borrar la llave del bastion
ssh -i "$KEY" -o StrictHostKeyChecking=no ${BASTION_USER}@${BASTION_IP} "shred -u ~/.labkey.pem 2>/dev/null || rm -f ~/.labkey.pem"
```

La evidencia queda en `~/forensics/` del bastión. Para el respaldo inmutable (snapshot EBS), ver el steering **chain-of-custody**.
