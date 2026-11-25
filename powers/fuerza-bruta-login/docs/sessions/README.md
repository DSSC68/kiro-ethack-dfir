# sessions/ — sesiones de prueba con Kiro

Artefacto_Documentacion obligatorio (Req 15.3). Aquí va la **conversación
exportada de Kiro** correspondiente a la generación e iteración de este
Power (`fuerza-bruta-login`): el historial de la sesión de chat/agente,
exportado tal cual, no un resumen.

## Qué poner aquí

- Exporta la sesión de Kiro (chat completo de la conversación en la que se
  generó/iteró este Power) y guárdala en este directorio, p. ej.
  `session-<fecha>.md` o el formato que produzca el exportador de Kiro.
- Si hubo más de una sesión relevante (por ejemplo, una para la generación
  inicial y otra para una corrección posterior), añade un archivo por
  sesión con un nombre que indique el orden o la fecha.

## Relación con el resto de `docs/`

- El prompt principal que inició cada sesión ya está resumido en
  `../prompt.md`.
- La realimentación y las correcciones derivadas de revisar el resultado de
  cada sesión están en `../feedback.md`.
- Este directorio complementa a los dos anteriores con la traza completa de
  la conversación, para que el Líder Técnico pueda verificar el proceso sin
  depender solo del resumen.

El archivo `.gitkeep` se mantiene hasta que se añada la primera sesión
exportada.
