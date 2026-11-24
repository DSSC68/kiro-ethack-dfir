# Cadena de custodia, respaldo inmutable y reporte

Reglas de preservación, el respaldo con snapshots EBS y la plantilla del reporte forense.

---

## Principios de preservación

- **Orden de volatilidad.** Captura lo efímero antes que lo estable: procesos y conexiones del contenedor → `docker diff` → rootfs/BD/logs.
- **No alterar la escena.** Escribe la evidencia fuera del contenedor. No instales herramientas dentro del contenedor durante la adquisición.
- **Integridad demostrable.** Todo artefacto entra en `MANIFEST.sha256`; el paquete lleva su propio hash. Se verifica tras cada copia.
- **Registro de acciones.** Cada operación (adquisición, transferencia, snapshot, borrado de llave) queda anotada con hora UTC y operador.
- **Dos puntos en el tiempo.** Si se analizó en caliente y luego se adquirió, son instantes distintos: decláralo.

## Archivo de custodia (incluido en el paquete)

Cada adquisición lleva `00-ACQUISITION-INFO.txt` en la raíz del paquete:

```
Adquisicion forense en caliente - Contenedor <nombre>
Fecha (UTC)  : <ISO-8601>
Host         : <hostname> / <ip>
Operador     : <usuario> (via bastion SSH)
Imagen       : <imagen:tag>
Metodo       : docker export + mysqldump + copia de logs/sesiones/uploads + timeline
Nota         : Laboratorio autorizado. Contenedor en ejecucion (live response).
```

### Bitácora de custodia (recomendada)

Mantén una tabla con cada traspaso:

| Fecha UTC | Acción | Origen → Destino | Hash (SHA256) | Operador |
|---|---|---|---|---|
| 2026-11-19T19:29Z | Adquisición | contenedor → /var/tmp (host) | `25f5…b87` | ec2-user |
| 2026-11-19T19:32Z | Transferencia | host → bastión (VPC) | `25f5…b87` (verificado) | analista |
| 2026-11-19T19:4xZ | Respaldo | bastión → snapshot EBS | `snap-0f67…b3b` | analista |

## Respaldo inmutable: snapshot EBS

El snapshot del volumen donde vive la evidencia (la instancia bastión) es la copia maestra: barata, inmutable y persiste aunque se destruya el laboratorio.

```bash
# 1. Volumen raíz del bastión
VOL=$(aws ec2 describe-volumes --profile <perfil> --region <region> \
  --filters "Name=attachment.instance-id,Values=<INSTANCE_ID_BASTION>" \
  --query 'Volumes[0].VolumeId' --output text)

# 2. Snapshot etiquetado
aws ec2 create-snapshot --profile <perfil> --region <region> \
  --volume-id "$VOL" \
  --description "Evidencia forense contenedor <fecha>" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=forensics-<fecha>},{Key=Lab,Value=<lab>}]' \
  --query '{Snap:SnapshotId,State:State}' --output json

# 3. Esperar a que complete antes de dar de baja el lab
aws ec2 wait snapshot-completed --profile <perfil> --region <region> --snapshot-ids <SNAP_ID>
```

Una vez `completed`, puedes destruir el entorno (las instancias dejan de facturar) conservando el snapshot. Para restaurar, registra una AMI desde el snapshot y relánzala.

> Si usas un `teardown.sh` que hace `export AWS_PROFILE` sin asignarlo, pásalo en el entorno: `AWS_PROFILE=<perfil> ./teardown.sh`.

## Plantilla de reporte forense

```markdown
# Análisis forense en caliente — Contenedor <nombre>

**Fecha del análisis:** <ISO UTC>
**Entorno:** <laboratorio autorizado>
**Método:** Live response vía bastión SSH (ProxyCommand). No se trató el host como objetivo.

## Entorno
| Recurso | Valor |
|---|---|
| Bastión | <id / ip> |
| Host del contenedor | <id / ip priv> |
| Contenedor | <nombre> (<imagen>), Id <...>, Started <...> |
| Red / SG | <subred, reglas de exposición> |

## Veredicto
<Una frase: qué fases alcanzaron y si hubo compromiso.>

## Evidencia recolectada
| Vector | Resultado | Interpretación |
|---|---|---|
| docker diff | ... | ... |
| access.log (IPs externas) | ... | ... |
| Peticiones sospechosas | ... | ... |
| error.log | ... | ... |
| BD (users/guestbook) | ... | ... |
| uploads | ... | ... |

## Línea de tiempo
1. <hora> — <evento>
2. ...

## Conclusión
<Estado del contenedor vs. inicialización; evidencia de compromiso sí/no; enlace al manifiesto y al snapshot.>
```

## Cierre del engagement

- [ ] Paquete verificado en el bastión (`MANIFEST.sha256` OK).
- [ ] Snapshot EBS `completed` de la evidencia.
- [ ] Reporte redactado con veredicto y evidencia enlazada.
- [ ] Llave `.pem` temporal borrada del bastión (`shred -u`).
- [ ] Laboratorio dado de baja (opcional) conservando snapshots.
- [ ] Bitácora de custodia actualizada.
