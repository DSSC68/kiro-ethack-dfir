---
inclusion: manual
---

# Límites anti-DoS

## Qué dice el apunte

El apunte (sección 06 "Cómo prevenir la fuerza bruta") lista las medidas de
defensa y evalúa si habrían frenado el ataque de su caso:

| Medida | Qué hace | ¿Habría frenado este ataque? |
|---|---|---|
| Contraseñas seguras | Largas, únicas, que no sean de fábrica | Sí: el atacante adivinó la contraseña por defecto |
| 2FA | Pide un código o confirmación además de la contraseña | Sí, aunque acertara la contraseña |
| Monitorear inicios de sesión | Alerta ante muchos intentos fallidos | Lo habría detectado a tiempo |
| Cambios frecuentes de contraseña | Obliga a renovarla cada cierto tiempo | En parte: forzaría cambiar la de fábrica |
| No reutilizar contraseñas | Bloquea usar contraseñas anteriores | Ayuda contra credential stuffing |
| **Limitar los intentos** | Bloquea la cuenta tras X intentos fallidos | **Sí: corta la fuerza bruta de raíz** |

El apunte recomienda 2FA como la medida más fuerte (sigue protegiendo
aunque la contraseña se adivine o se filtre) y, como segunda opción,
limitar los intentos, porque ataca directamente la mecánica de la fuerza
bruta.

## Regla obligatoria (anti-DoS para el operador de este Power)

Si el objetivo cuenta con "Limitar los intentos" (la medida que el apunte
señala como la que corta la fuerza bruta de raíz), un ataque con
concurrencia alta o sin pausas no solo fallará: puede bloquear cuentas
legítimas o saturar el servicio, convirtiendo la prueba en una denegación de
servicio no autorizada. Por eso:

1. **`script-kiddie` (default)**: concurrencia mínima, diccionario acotado
   (ver `preparacion-diccionario.md`). Objetivo: confirmar si la falla
   "contraseña de fábrica sin control de intentos" existe, sin generar
   carga.
2. **`green-hat`**: concurrencia moderada, diccionario más amplio, con
   pausas entre tandas si el objetivo responde con bloqueos o latencia
   creciente.
3. **`pro`**: el operador puede ajustar manualmente la concurrencia, pero
   sigue siendo responsable de no exceder lo acordado en el `scope.yaml`
   del engagement (ventanas de prueba, límites de tasa pactados con el
   cliente).
4. Ante cualquier señal de bloqueo de cuenta, lista negra de IP, o
   degradación del servicio objetivo, **detener la ejecución** y documentar
   el hallazgo: esa reacción del objetivo es, en sí misma, evidencia de que
   el control "Limitar los intentos" (o un WAF/IPS) está funcionando.

## Parámetros de Hydra para limitar la velocidad (añadido)

El apunte no da parámetros de herramienta. Lo siguiente se agrega para
hacer operable el límite:

```bash
# (añadido) script-kiddie: 1-4 tareas en paralelo
hydra -L users.txt -P passwords.txt -t 4 <target> http-post-form "..."

# (añadido) green-hat: concurrencia moderada + espera entre reintentos de conexión
hydra -L users.txt -P passwords.txt -t 8 -w 30 <target> http-post-form "..."

# (añadido) Limitar intentos totales en pruebas tipo "pro" acotadas (ejemplo ilustrativo)
# -t = tareas en paralelo, -w = espera (segundos) por respuesta antes de reintentar
hydra -L users.txt -P passwords.txt -t 16 -w 30 <target> http-post-form "..."
```

Valores de `-t` sugeridos, no absolutos: ajustar siempre contra lo acordado
en el engagement y detener si el objetivo muestra señales de degradación.
