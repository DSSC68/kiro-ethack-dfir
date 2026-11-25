---
inclusion: manual
---

# Interpretación de resultados

## Qué dice el apunte

El apunte (sección "Cómo se vería una fuerza bruta (no viene en tu log)")
da el patrón de referencia tal como aparece en los registros del servidor:
muchos intentos fallidos seguidos, desde la misma IP, contra la misma
cuenta, y al final uno exitoso. Ejemplo ilustrativo citado en el apunte:

```
10:02:11 FAILED login user=admin from 198.51.100.44
10:02:12 FAILED login user=admin from 198.51.100.44
10:02:12 FAILED login user=admin from 198.51.100.44
... cientos más ...
10:09:47 SUCCESS login user=admin from 198.51.100.44
```

El apunte aclara que esta vista es del lado del **servidor** (logs de
autenticación); la captura de `tcpdump` del ejercicio es del lado del
**visitante**, por lo que no muestra el ataque al panel directamente — solo
sus consecuencias (el sitio modificado, la redirección).

## Regla obligatoria

1. Al revisar la salida de Hydra, el resultado relevante es la línea con el
   par usuario/contraseña válido que Hydra reporta como encontrado
   (equivalente al `SUCCESS` del patrón de log del apunte).
2. Documentar el hallazgo con el mismo vocabulario que usaría Blue Team /
   Digital Forensics al leer un log de autenticación: cuenta objetivo, IP de
   origen, cantidad de intentos fallidos antes del éxito, y marca de tiempo.
   Esto es lo que debe quedar en el `producesActionLog` del Power.
3. Si Hydra no encuentra ninguna credencial válida con el diccionario usado,
   el resultado también se documenta: confirma (parcialmente) que el
   control de "contraseña distinta a la de fábrica" sí está en su lugar, lo
   cual es información útil para el informe de GRC/VA.
4. No continuar automáticamente hacia explotación tras obtener la
   credencial (cambiar contraseñas, subir archivos, etc.): el apunte ubica
   esas acciones en la fase posterior del caso (Maintaining Access /
   Covering Tracks), fuera del alcance de este Power.

## Mapeo de la salida de Hydra al patrón del apunte (añadido)

El apunte no describe el formato de salida de Hydra. El siguiente mapeo se
agrega para traducir la salida real de la herramienta al patrón de log que
el apunte sí documenta:

```
# (añadido) Salida típica de Hydra al encontrar una credencial válida:
[STATUS] attempt finished for 203.0.113.22 (waiting for children to finish)
[80][http-post-form] host: 203.0.113.22   login: admin   password: admin

# (añadido) Traducción al patrón de log del apunte, para el reporte:
<n_intentos_previos> FAILED login user=admin from <ip_atacante>
<timestamp_hallazgo> SUCCESS login user=admin from <ip_atacante>  # password=admin
```
