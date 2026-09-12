# Anexos técnicos — Prompts y JSON de referencia

Este documento reúne, a modo de respaldo técnico, los prompts de IA y ejemplos de JSON utilizados en cada etapa del flujo. No es un requisito de la consigna, pero se incluye para dar trazabilidad completa del diseño.

---

## A. Primer agente IA — interpretación de novedades

### A.1 System Prompt

📋 **[PEGAR AQUÍ: versión final del system prompt del primer agente]**

### A.2 Ejemplos de input / output

📋 **[PEGAR AQUÍ: JSON de novedad de entrada + JSON de salida — caso "ok" (ej. vacaciones)]**

📋 **[PEGAR AQUÍ: JSON de salida — caso despido (estado: requiere_revision)]**

📋 **[PEGAR AQUÍ: JSON de salida — caso alta de empleado (estado: requiere_revision)]**

---

## B. Catálogo de conceptos (Airtable → texto plano para RAG)

### B.1 Código de estructuración (nodo Code)

📋 **[PEGAR AQUÍ: código JavaScript del nodo que deduplica y aplana el catálogo de Airtable]**

### B.2 Ejemplo de catálogo resultante

📋 **[PEGAR AQUÍ: texto plano del catálogo tal como se lo pasa al agente]**

---

## C. Segundo agente IA — validación de corrección manual

### C.1 System Prompt

📋 **[PEGAR AQUÍ: versión final del system prompt del segundo agente]**

### C.2 Ejemplos de input / output

📋 **[PEGAR AQUÍ: JSON de pares enviados + JSON de resultado — caso corrección normal]**

📋 **[PEGAR AQUÍ: JSON de resultado — caso alta con datos completos]**

📋 **[PEGAR AQUÍ: JSON de resultado — caso alta con datos incompletos]**

---

## D. Código de orquestación (nodos Code de n8n)

### D.1 Extracción de novedades del mensaje de Telegram

📋 **[PEGAR AQUÍ: código del nodo que separa el mensaje en una novedad por empleado]**

### D.2 Aplanado de la salida del primer agente

📋 **[PEGAR AQUÍ: código del nodo que convierte el JSON del agente en filas campo/valor]**

### D.3 Armado del mensaje de revisión (HITL)

📋 **[PEGAR AQUÍ: código del nodo que agrupa casos por legajo y arma el mensaje de Telegram]**

### D.4 Parseo de la respuesta manual

📋 **[PEGAR AQUÍ: código del nodo que interpreta la respuesta de Telegram en pares campo/valor]**

### D.5 Agrupación del resultado de validación (previene multiplicación en el loop)

📋 **[PEGAR AQUÍ: código del nodo que agrupa los resultados del segundo agente en un único item por caso]**

### D.6 Armado de fila para Google Sheets

📋 **[PEGAR AQUÍ: código del nodo que arma el objeto final a escribir en Sheets, soportando tanto la rama automática como la de corrección]**

---

## E. Registro en Airtable (Log de Casos)

📋 **[PEGAR AQUÍ: JSON de ejemplo de un registro creado en la rama automática]**

📋 **[PEGAR AQUÍ: JSON de ejemplo de un registro creado en la rama HITL]**

---

## F. Aprendizajes técnicos del desarrollo

Documentados a modo de evidencia del proceso de prueba y depuración (relevante para el criterio de resiliencia):

- **Bug de multiplicación de items en el loop:** al no agrupar los resultados de la validación en un único item por caso antes de reingresar al `Loop Over Items`, cada campo corregido se trataba como un caso nuevo, multiplicando exponencialmente la cola pendiente. Resuelto agregando un nodo de agregación por legajo antes de retroalimentar el loop.
- **Formato de salida no determinístico del modelo:** en ciertos casos el modelo de IA devolvía texto libre en lugar de JSON estructurado pese a la instrucción explícita. Mitigado activando la validación de formato de salida (JSON Schema) a nivel de configuración del agente, no solo por instrucción de prompt.
- **Referencias entre nodos por posición vs. por nombre:** el uso de `.item` para referenciar nodos no conectados directamente en la cadena de ejecución generó errores de resolución; se reemplazó por `.first()` o por lectura directa del `$json` del item en curso, más robusto para flujos con bifurcaciones y loops.
