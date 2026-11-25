---
inclusion: manual
---

# Preparación del diccionario de usuarios/contraseñas

## Qué dice el apunte

El apunte (`docs/fuentes/896492b3-4_fuerza_bruta_ejercicio.pdf`, sección 02
"¿Qué es un ataque de fuerza bruta?") distingue cuatro variantes:

- **Fuerza bruta simple**: prueba combinaciones de una en una (`aaaa`, `aaab`…).
  Lento, pero eventualmente funciona con contraseñas cortas.
- **Ataque de diccionario**: prueba contraseñas comunes o por defecto
  (`admin`, `123456`, `password`). **Es lo que ocurrió en el caso del
  apunte**: la contraseña de administrador seguía siendo la de fábrica.
- **Credential stuffing**: usa usuario y contraseña filtrados de otro sitio,
  apostando a que la persona los repite.
- **Password spraying**: prueba una misma contraseña común contra muchas
  cuentas, para no activar bloqueos.

El apunte identifica las dos fallas que permitieron el acceso en su caso
(sección 01): (1) la contraseña de administrador seguía siendo la de
fábrica, y (2) no había ningún control contra fuerza bruta (sin límite de
intentos, sin 2FA, sin monitoreo de inicios de sesión).

## Regla obligatoria

Antes de ejecutar Hydra, construir el diccionario según el tipo de ataque
declarado para la prueba:

1. Por defecto (`script-kiddie`), priorizar el **ataque de diccionario**
   replicando el caso del apunte: credenciales de fábrica del
   servicio/dispositivo objetivo y contraseñas comunes conocidas
   (`admin`/`admin`, `admin`/`password`, `admin`/`123456`, etc.).
2. No usar fuerza bruta simple (combinatoria exhaustiva) salvo que el modo
   `pro` lo justifique explícitamente — el apunte la describe como lenta y
   solo viable contra contraseñas cortas; no es el patrón que motivó este
   Power.
3. No usar credential stuffing con credenciales filtradas de terceros sin
   autorización explícita y verificación de que esas credenciales
   corresponden al mismo propietario del objetivo en el `scope.yaml`.

## Comandos (añadido)

El apunte no incluye comandos de preparación de diccionarios. Lo siguiente
se agrega para que el Power sea operable, usando herramientas estándar de
Kali:

```bash
# (añadido) Diccionario mínimo de credenciales de fábrica/comunes
cat > users.txt <<EOF
admin
administrator
root
EOF

cat > passwords.txt <<EOF
admin
password
123456
EOF
```

```bash
# (añadido) Reusar las listas incluidas en Kali si existen (SecLists)
ls /usr/share/seclists/Passwords/Default-Credentials/ 2>/dev/null
```

Para `green-hat`/`pro`, ampliar `passwords.txt` con una lista de contraseñas
comunes más extensa (p. ej. `rockyou.txt` si está disponible en el entorno),
manteniendo el foco en el patrón de diccionario, no en combinatoria pura.
