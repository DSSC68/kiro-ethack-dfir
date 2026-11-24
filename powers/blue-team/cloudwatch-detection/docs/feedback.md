# feedback.md — cloudwatch-detection (Blue Team)

Artefacto de documentación (Req 15.1). Registro de aprendizajes y ajustes de la
Power tras cada uso.

## Formato por entrada

- **Fecha (UTC):**
- **Sesión / engagement:**
- **Qué funcionó:** (detecciones acertadas, consultas útiles)
- **Falsos positivos / negativos:** (p. ej. health checks contados como ataque)
- **Ajuste propuesto:** (umbral, patrón de parseo, nueva consulta)

## Entradas

- 2026-11-20 — Alta de la Power (línea base docente). Detección de scanning web,
  fuerza bruta y webshell sobre `/ethack-lab/dvwa` validada contra el stream real
  del contenedor DVWA. Pendiente de los alumnos: afinar umbrales y añadir
  detección de SQLi/XSS por payload y correlación con VPC Flow Logs (capa de red).
