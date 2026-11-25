---
inclusion: manual
---

# Medidas de prevención y hardening

## Qué dice el apunte

El apunte (sección "Cómo prevenir la fuerza bruta") da la tabla completa de
medidas, qué hace cada una, y si habría frenado el ataque de su caso:

| Medida | Qué hace | ¿Habría frenado este ataque? |
|---|---|---|
| Contraseñas seguras | Largas, únicas, que no sean de fábrica | Sí: el atacante adivinó la contraseña por defecto |
| 2FA (autenticación de dos factores) | Además de la contraseña, pide un código o confirmación | Sí, aunque acertara la contraseña |
| Monitorear inicios de sesión | Alerta ante muchos intentos fallidos | Lo habría detectado a tiempo |
| Cambios frecuentes de contraseña | Obliga a renovarla cada cierto tiempo | En parte: forzaría cambiar la de fábrica |
| No reutilizar contraseñas | Bloquea usar contraseñas anteriores | Ayuda contra credential stuffing |
| Limitar los intentos | Bloquea la cuenta tras X intentos fallidos | Sí: corta la fuerza bruta de raíz |

El apunte recomienda explícitamente: elegir **2FA** como primera medida —
"es la más fuerte porque sigue protegiendo incluso si la contraseña se
adivina o se filtra". Como segunda opción, **limitar los intentos** —
"ataca justo la mecánica de la fuerza bruta".

El apunte también identifica las dos fallas concretas de control en su caso
(sección "Qué pasó"): la contraseña de administrador seguía siendo la de
fábrica, y no había ningún control contra fuerza bruta (ni límite de
intentos, ni 2FA, ni monitoreo de inicios de sesión).

## Regla obligatoria

1. Al reportar un hallazgo de fuerza bruta confirmado
   (`patron-fuerza-bruta-logs.md` + `distincion-casos.md`), verificar y
   documentar explícitamente si cada una de las seis medidas de la tabla
   está presente o ausente en el objetivo.
2. La recomendación principal del reporte debe priorizar **2FA** sobre las
   demás medidas, siguiendo el orden que da el apunte, y **limitar los
   intentos** como segunda recomendación si 2FA no es viable de inmediato.
3. No recomendar medidas fuera de esta tabla sin marcarlas como
   `(añadido)` — el apunte no cubre, por ejemplo, WAF, listas de bloqueo de
   IP, ni CAPTCHA.

## Verificación práctica (añadido)

El apunte no da comandos para verificar si estas medidas están activas. Se
agrega lo siguiente como punto de partida de verificación manual:

```bash
# (añadido) Buscar evidencia de que SÍ hay límite de intentos: bloqueos o
# silencios tras varios FAILED en el log de autenticación
grep -A2 "FAILED login" auth.log | grep -i "locked\|blocked\|too many attempts"

# (añadido) Buscar evidencia de monitoreo activo (alertas ya generadas)
grep -i "alert\|threshold exceeded" auth.log

# (añadido) Confirmar ausencia de límite de intentos: contar cuántos FAILED
# consecutivos hubo antes del SUCCESS para la misma cuenta/IP (ver
# patron-fuerza-bruta-logs.md); un número alto sin bloqueo intermedio es
# evidencia de que la medida "Limitar los intentos" no estaba activa.
```

## Reporte del hallazgo

El reporte final de este Power debe reconstruir la cadena completa del
apunte (sección "Qué pasó: la historia completa"): fuerza bruta contra el
panel → modificación del código del sitio → cambio de contraseña de admin
(bloqueo del dueño legítimo) → redirección de visitantes al sitio falso →
descubrimiento por quejas de clientes. Cada eslabón de esa cadena que se
confirme con evidencia real (log o captura) debe citarse con su fase
correspondiente, y las medidas de prevención recomendadas deben
corresponder a la falla específica que permitió ese eslabón.
