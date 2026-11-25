---
inclusion: manual
---

# Leer tcpdump por capas TCP/IP

## Qué dice el apunte

El apunte (sección "¿Qué protocolo está involucrado? Las 4 capas de
TCP/IP") da la tabla exacta para ubicar cada protocolo observado en
`tcpdump`:

| Capa | Qué se ve en `tcpdump` |
|---|---|
| 4. Aplicación | HTTP (`.http`, puerto 80, `GET / HTTP/1.1`) y DNS (`.domain`, puerto 53, `A?`). Aquí vive la respuesta. |
| 3. Transporte | TCP (`Flags [S]`, `[S.]`, `[.]`, `[P.]`) para HTTP, y UDP para DNS. |
| 2. Internet | IP: la palabra `IP` al inicio de cada línea. |
| 1. Acceso a la red | Ethernet/Wi-Fi. `tcpdump` no lo muestra en estas líneas. |

El apunte también da la lectura línea por línea del caso, fase por fase
(sección "Cómo leer tu log, línea por línea"), usando `your.machine` como la
computadora del sandbox y `dns.google` como servidor DNS:

- **Fase 1 — DNS para el sitio legítimo** (`14:18:32`):
  ```
  14:18:32.192571 IP your.machine.52444 > dns.google.domain: 35084+ A? yummyrecipesforme.com. (24)
  14:18:32.204388 IP dns.google.domain > your.machine.52444: 35084 1/0/0 A 203.0.113.22 (40)
  ```
  Pregunta de referencia del apunte: "¿qué IP tiene yummyrecipesforme.com?".
  Respuesta: `203.0.113.22`. DNS funcionando normalmente.

- **Fase 2 — Conexión al sitio y descarga** (`14:18:36`):
  ```
  14:18:36.786501 IP your.machine.36086 > yummyrecipesforme.com.http: Flags [S] ...
  14:18:36.786517 IP yummyrecipesforme.com.http > your.machine.36086: Flags [S.] ...
  14:18:36.786529 IP your.machine.36086 > yummyrecipesforme.com.http: Flags [.] ...
  14:18:36.786589 IP your.machine.36086 > yummyrecipesforme.com.http: Flags [P.] ... length 73: HTTP: GET / HTTP/1.1
  ```
  Handshake completo (`S` → `S.` → `.`) al puerto 80, y el navegador pide la
  página con `GET`. Después hay mucho tráfico en el puerto 80 durante ~2
  minutos: la página cargando y la descarga del archivo malicioso.

- **Fase 3 — El cambio sospechoso** (`14:20:32`):
  ```
  14:20:32.192571 IP your.machine.52444 > dns.google.domain: 21899+ A? greatrecipesforme.com. (24)
  14:20:32.204388 IP dns.google.domain > your.machine.52444: 21899 1/0/0 A 192.0.2.17 (40)
  ```
  Clave del caso: nadie escribió `greatrecipesforme.com`, lo pidió el
  malware que se acababa de ejecutar. Dominio nuevo, casi idéntico al
  original, con otra IP (`192.0.2.17`).

- **Fase 4 — Conexión al sitio falso** (`14:25:29`):
  ```
  14:25:29.576493 IP your.machine.56378 > greatrecipesforme.com.http: Flags [S] ...
  14:25:29.576510 IP greatrecipesforme.com.http > your.machine.56378: Flags [S.] ...
  14:25:29.576524 IP your.machine.56378 > greatrecipesforme.com.http: Flags [.] ...
  14:25:29.576590 IP your.machine.56378 > greatrecipesforme.com.http: Flags [P.] ... HTTP: GET / HTTP/1.1
  ```
  Mismo patrón que la fase 2, pero contra el sitio falso y desde un puerto
  de origen nuevo (`56378`). El navegador ya está en la página con malware.

El apunte advierte una discrepancia a tener en cuenta en el informe: el log
dice `192.0.2.17`, pero alguna guía de referencia dice `192.0.2.172` — en el
informe debe usarse la IP que aparece en el log real, porque es la
evidencia.

## Regla obligatoria

1. Antes de concluir qué protocolo "carga el problema", clasificar cada
   línea de la captura en su capa usando la tabla de arriba.
2. En un caso con el mismo patrón que el del apunte, el protocolo que carga
   el problema es **HTTP** (capa de Aplicación): es el que pide la página
   comprometida, descarga el malware, y después carga el sitio falso. DNS
   participa pero solo resuelve nombres — funciona como debía en ambas
   resoluciones (la legítima y la del dominio falso).
3. Usar siempre la IP y los timestamps tal como aparecen en el log real
   capturado, no los de una guía o plantilla de referencia.

## Comandos de lectura (añadido)

El apunte no da comandos de `tcpdump`/`tshark` para producir estas líneas;
lo siguiente se agrega para filtrar una captura real:

```bash
# (añadido) Leer una captura ya guardada, mostrando solo tráfico DNS y HTTP
tcpdump -r captura.pcap -n port 53 or port 80

# (añadido) Aislar solo las resoluciones DNS (capa de Aplicación / DNS)
tcpdump -r captura.pcap -n port 53

# (añadido) Aislar solo el handshake y tráfico HTTP (capas Transporte/Aplicación)
tcpdump -r captura.pcap -n port 80
```
