# sessions/ — sesiones de prueba con Kiro

Artefacto_Documentacion obligatorio (Req 15.3). Aquí va la **conversación
exportada de Kiro** correspondiente a la generación e iteración de este
Power (`deteccion-fuerza-bruta`): el historial de la sesión de chat/agente,
exportado tal cual, no un resumen.

## Qué poner aquí

- Exporta la sesión de Kiro (chat completo de la conversación en la que se
  generó/iteró este Power) y guárdala en este directorio, p. ej.
  `session-<fecha>.md` o el formato que produzca el exportador de Kiro.
- Este Power tuvo al menos dos momentos relevantes: la generación inicial
  del Power de detección, y la corrección posterior del contrato
  `x-ethack` (asignación de `forensicsStage: "Detect"` para cumplir la
  regla XOR y pasar `make validate`). Si se exportan por separado, nombrar
  los archivos de forma que el orden quede claro.

## Relación con el resto de `docs/`

- Los prompts que iniciaron cada momento ya están resumidos en
  `../prompt.md`.
- La realimentación y las correcciones derivadas (incluida la del contrato
  `x-ethack`) están en `../feedback.md`.
- Este directorio complementa a los dos anteriores con la traza completa de
  la conversación, para que el Líder Técnico pueda verificar el proceso sin
  depender solo del resumen.

El archivo `.gitkeep` se mantiene hasta que se añada la primera sesión
exportada.
