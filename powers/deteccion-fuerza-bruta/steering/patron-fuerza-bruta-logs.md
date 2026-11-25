---
inclusion: manual
---

# Identificar el patrón de fuerza bruta en logs

## Qué dice el apunte

El apunte (sección "Cómo se vería una fuerza bruta (no viene en tu log)")
es explícito: la captura de `tcpdump` del ejercicio es del lado del
**visitante**, así que no muestra el ataque al panel de administración. En
los registros del **servidor**, una fuerza bruta se ve así: muchos intentos
fallidos seguidos, desde la misma IP, contra la misma cuenta, y al final uno
exitoso. Ejemplo ilustrativo citado literalmente en el apunte:

```
10:02:11 FAILED login user=admin from 198.51.100.44
10:02:12 FAILED login user=admin from 198.51.100.44
10:02:12 FAILED login user=admin from 198.51.100.44
... cientos más ...
10:09:47 SUCCESS login user=admin from 198.51.100.44
```

El apunte también da el contexto del caso completo (sección "Qué pasó: la
historia completa"): un exempleado prueba contraseñas por defecto conocidas
hasta atinarle; entra porque la contraseña de admin nunca se cambió y no
había límite de intentos.

## Regla obligatoria

1. El patrón que confirma fuerza bruta en un log de autenticación tiene
   **tres rasgos simultáneos**, tal como los da el apunte:
   - misma IP de origen en todos los intentos,
   - misma cuenta objetivo,
   - secuencia de múltiples `FAILED` terminando en un `SUCCESS`.
   Si falta cualquiera de los tres, no se puede afirmar fuerza bruta solo
   con el log; hay que seguir investigando (p. ej. password spraying usa la
   misma contraseña contra *muchas* cuentas, no el patrón de arriba).
2. Registrar también el intervalo de tiempo entre el primer `FAILED` y el
   `SUCCESS` final (en el ejemplo del apunte: de `10:02:11` a `10:09:47`),
   porque es la evidencia de que no hubo límite de intentos ni bloqueo
   temporal — una de las dos fallas de control que señala el apunte.

## Comandos de búsqueda (añadido)

El apunte no da comandos de análisis de log; lo siguiente se agrega usando
`grep`/`awk` sobre un log de autenticación con el formato del ejemplo:

```bash
# (añadido) Contar intentos FAILED por IP y usuario
grep "FAILED login" auth.log | awk '{print $NF, $(NF-2)}' | sort | uniq -c | sort -rn

# (añadido) Ver la secuencia completa (FAILED...SUCCESS) para una IP sospechosa
grep "198.51.100.44" auth.log

# (añadido) Confirmar que una IP con muchos FAILED termina en SUCCESS
grep "198.51.100.44" auth.log | grep -E "FAILED|SUCCESS"

# (añadido) Medir el intervalo entre el primer FAILED y el SUCCESS (manual,
# comparando las marcas de tiempo de la primera y última línea del filtro anterior)
```
