---
inclusion: manual
---

# Distinguir DNS caído / SYN flood / sitio comprometido

## Qué dice el apunte

El apunte (sección "Cómo distinguir este caso de los anteriores") da la
tabla comparativa completa, construida sobre las mismas preguntas guía para
los tres casos que cubre la serie de ejercicios:

| Pregunta | DNS caído | SYN flood | Sitio comprometido |
|---|---|---|---|
| ¿Hay errores en el log? | Sí: `udp port 53 unreachable` | Sí: `504`, `RST`, y luego silencio | No. Todo conecta bien |
| ¿Se completa el handshake? | Ni se llega a TCP | No para el atacante | Sí, siempre |
| ¿Qué llama la atención? | ICMP en vez de respuesta DNS | Volumen de SYN desde una IP | Un dominio nuevo que nadie pidió |
| ¿Qué protocolo carga el problema? | DNS / UDP | TCP | HTTP |
| ¿Qué falló? | Un servicio | La disponibilidad | Una contraseña y el control de acceso |

El apunte resume la regla de oro de este caso así: si el log se ve "sano"
pero el navegador termina en un dominio diferente, sospechar de redirección
maliciosa. Buscar una segunda consulta DNS, sin que el usuario la pidiera,
hacia un nombre parecido al original (typosquatting).

## Regla obligatoria

1. Antes de clasificar un incidente como "sitio comprometido por fuerza
   bruta", confirmar las cinco respuestas de la columna correspondiente:
   sin errores de log, handshake completo, dominio nuevo no solicitado,
   HTTP como protocolo que carga el problema, y la falla ubicada en
   contraseña + control de acceso (no en disponibilidad ni en DNS).
2. Si aparecen errores tipo `udp port 53 unreachable` o respuestas ICMP en
   vez de DNS, el caso es **DNS caído**, no fuerza bruta — no continuar con
   este Power.
3. Si aparece un volumen alto de `SYN` desde una IP sin que el handshake se
   complete para el atacante, el caso es **SYN flood**, no fuerza bruta —
   no continuar con este Power.
4. Solo cuando el patrón coincide con la columna "Sitio comprometido" se
   procede a aplicar `patron-fuerza-bruta-logs.md` y
   `prevencion-hardening.md`.

## Checklist de clasificación (añadido)

El apunte da la tabla pero no un procedimiento paso a paso para aplicarla.
Se agrega este checklist operativo:

```text
# (añadido) Checklist de triage, en orden:
1. ¿El log de red muestra "udp port 53 unreachable" o ICMP en vez de A record?
   -> Sí: DNS caído. Detener triage.
2. ¿Hay un volumen alto de [S] desde una misma IP sin [S.] de respuesta exitosa
   para el atacante, seguido de silencio/RST?
   -> Sí: SYN flood. Detener triage.
3. ¿El handshake se completa siempre y el log de red se ve "sano", pero aparece
   una segunda consulta DNS no solicitada por el usuario hacia un dominio
   parecido al legítimo?
   -> Sí: sospechar sitio comprometido por fuerza bruta + typosquatting.
      Continuar con patron-fuerza-bruta-logs.md.
```
