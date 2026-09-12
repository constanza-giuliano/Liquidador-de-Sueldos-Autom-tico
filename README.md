# Liquidador de Sueldos Automático — UTHGRA
### Proyecto final — Curso de Automatización con IA

Sistema de automatización que interpreta novedades mensuales de liquidación de sueldos (enviadas en lenguaje natural por Telegram), las valida contra el convenio colectivo UTHGRA, actualiza automáticamente la base de cálculo en Google Sheets, y gestiona un ciclo de aprobación humana (HITL) para los casos que requieren revisión: despidos, altas de personal y correcciones de datos.

---

## 1. Caso de uso

Estudio contable que liquida sueldos bajo el convenio UTHGRA-CATC para un cliente con 117 empleados bajo un único convenio. Actualmente el proceso de traducir novedades (vacaciones, licencias, ART, altas, bajas) a la planilla de cálculo se hace a mano. Este sistema automatiza esa traducción, dejando solo la aprobación humana en los puntos de mayor riesgo (despidos, altas, datos ambiguos).

---


## 2. Entregable 1 — Mapa de arquitectura (20%)

📄 [Ver diagrama de arquitectura](ANEXOS/arquitectura.pdf)
📄 [Ver flujo de n8n](ANEXOS/flujo_n8n_completo.png)

**Resumen del flujo:**
1. El estudio envía las novedades del mes por Telegram, en texto libre, para todos los empleados en un solo mensaje.
2. Un nodo de código separa el mensaje en una novedad por empleado.
3. Un primer agente de IA interpreta cada novedad, la compara contra un catálogo de conceptos válidos (Airtable, actúa como fuente de verdad tipo RAG), y determina qué campos corresponde modificar.
4. Casos con datos completos y sin ambigüedad → se escriben automáticamente en Google Sheets (motor de cálculo con fórmulas).
5. Casos de despido, alta de personal, datos incompletos o conceptos no reconocidos → entran a una cola de revisión humana (Loop Over Items).
6. Por cada caso pendiente, el sistema envía un mensaje por Telegram pidiendo la corrección o los datos faltantes, y espera la respuesta (nodo *Send and Wait*).
7. La respuesta pasa por un segundo agente de IA que valida formato, normaliza datos (fechas, texto) y confirma si el caso queda resuelto.
8. Si queda resuelto: se escribe en Sheets (Update o Append según corresponda) y el sistema pasa al siguiente caso pendiente.
9. Si no queda resuelto: se reintenta el mismo caso, pidiendo puntualmente lo que falta.
10. Cuando no quedan casos pendientes, se envía un mensaje de cierre.
11. Cada intento (automático o manual) queda registrado en una tabla de Airtable ("Log de Casos"), con su estado, para trazabilidad y control de errores.

---

## 2. Entregable 2 — Estructuras de datos documentadas (20%)

### 5.1 Google Sheets — motor de cálculo

| Hoja 1| Captar todas las novedades relativas a cada empleado que forma parte de la nomina de la empresa |
|---|---|
| Datos Empresa | Datos fijos del empleador |
| Escala/Convenio | Valores por categoría (básico, no remunerativo) según el acuerdo UTHGRA vigente |
| Empleados | Datos de cada empleado + columnas de días (editadas por el sistema) + columnas de resultado (calculadas por fórmula, nunca editadas por la IA) |

**Regla de diseño clave:** la IA únicamente escribe en las columnas de "días" y datos estructurales de alta. Los campos de resultado (Neto, Bruto, Aportes, Contribuciones) siempre se calculan por fórmula — la IA nunca hace aritmética de montos.

🔗 https://docs.google.com/spreadsheets/d/17Huq61I99lb1DyzJ_HmtNzGLsm7dj8Qss43D3zzx9Ig/edit?usp=sharing

### 5.2 Airtable 
#### Memoria y registro del sistema

**Tabla "Conceptos_liquidacion.xlsx"** (catálogo cerrado, actúa como fuente RAG):

📋 JSON CORTO DE EJEMPLO:
[
  {
    "Concepto": "Dias ART 10",
    "Descripcion": "Primeros 10 Dias en los que no trabajo por tener licencia por accidente de trabajo. Comienza el dia 1 del accidente de trabajo y se extiende hasta maximo el dia numero 10 desde la fecha de inicio",
    "Valor aceptado": "Numero entero del 0 al 10"
  }]


🔗 https://airtable.com/appqY1jYtEzBGJRrb/shrAiUMsPblmUqziY

#### Tabla "Log de Casos"** (registro de trazabilidad):

| Campo | Descripción |
|---|---|
| Legajo | Empleado afectado |
| Periodo | Mes liquidado |
| Rama | Automática / HITL |
| Estado | ok / requiere_revision / error |
| Campos | Detalle de qué se cambió |
| Observación | Motivo, si hubo revisión |
| Texto original | Novedad tal cual se recibió |

📋 JSON CORTO DE EJEMPLO:
[{
    "Legajo": "00025",
    "Periodo": "Agosto 2026",
    "Rama": "Automática",
    "Estado": "ok",
    "Campo": "Dias Vacaciones",
    "Observacion": "",
    "Texto original": "tuvo vacaciones del 08/08 al 25/08",
    "Intento": 1,
    "Fecha": "9/12/2026 9:16pm"
  }]

🔗 https://airtable.com/app4ap36WRIaXrZn3/shrFirxxI2lcSCHvD


### 5.3 Esquemas JSON de las integraciones

**Salida del primer agente IA (interpretación de novedad):**

📋 [
  {
    "output": "{\n  \"legajo\": \"00025\",\n  \"periodo\": \"Agosto 2026\",\n  \"texto_original\": \"tuvo vacaciones del 08/08 al 25/08\",\n  \"estado\": \"ok\",\n  \"motivo_revision\": \"\",\n  \"campos_modificados\": [\n    { \"campo\": \"Dias Vacaciones\", \"valor\": \"18\" },\n    { \"campo\": \"Dias Trabajados\", \"valor\": \"12\" }\n  ]\n}"
  },
  {
    "output": "{\n  \"legajo\": \"00101\",\n  \"periodo\": \"Agosto 2026\",\n  \"texto_original\": \"fue despedido sin causa el 10/8/2026\",\n  \"estado\": \"requiere_revision\",\n  \"motivo_revision\": \"Desvinculación detectada, requiere validación humana antes de liquidar\",\n  \"campos_modificados\": []\n}"
  },
  {
    "output": "{\n  \"legajo\": \"00203\",\n  \"periodo\": \"Agosto 2026\",\n  \"texto_original\": \"fue dado de alta el 17/08/2026\",\n  \"estado\": \"requiere_revision\",\n  \"motivo_revision\": \"Alta de empleado nuevo, requiere carga manual de datos estructurales antes de procesar: Legajo, Nombre, Categoria, Jornada, FI, Sit.Contrib, Dias trabajados\",\n  \"campos_modificados\": []\n}"
  }
]

**Salida del segundo agente IA (validación de corrección manual):**
Ejemplo para el segundo caso que general el primer agente:
[
  {
    "output": "00101, Dias Trabajados, 10; *Dias Vac NG Prop, 7; *Dias SAC Prop, 66"
  }
]

*(El detalle completo de todos los JSON de prueba usados durante el desarrollo está en `ANEXOS.md`.)*

---

## 6. Entregable 3 — Optimización de costos (20%)

### 6.1 Justificación de modelos por tarea

El sistema usa **dos modelos de IA distintos**, elegidos según la naturaleza de cada tarea — no por preferencia arbitraria, sino porque cada una tiene requisitos distintos de razonamiento y de frecuencia de uso.

| Tarea | Modelo | Por qué |
|---|---|---|
| **Interpretación de novedades** (primer agente) | **Claude** | Tarea de comprensión de lenguaje natural con reglas implícitas complejas: cálculo de tramos de ART (día 10 vs. día 11), detección de intención (despido vs. renuncia vs. alta), inferencia de Días Trabajados a partir de otros campos. Requiere razonamiento en varios pasos y buena adherencia a instrucciones normativas — se ejecuta **una vez por novedad y por mes**, volumen bajo, así que el costo por llamada tiene bajo impacto total. |
| **Validación de correcciones manuales** (segundo agente) | **DeepSeek** | Tarea más mecánica: matchear texto contra un catálogo cerrado, normalizar formatos de fecha y texto, chequear rangos válidos. No requiere el mismo nivel de razonamiento normativo. Se ejecuta **potencialmente varias veces por caso** (reintentos dentro del loop HITL), por lo que el volumen de llamadas es mayor y variable — un modelo de menor costo por token reduce el impacto de los reintentos sin sacrificar calidad, ya que la tarea es de clasificación/validación, no de interpretación abierta. |

### 6.2 Estimación de costos (valores de referencia)

> Nota: los precios de las APIs de IA cambian con frecuencia. Los valores siguientes son órdenes de magnitud a fines del análisis comparativo del proyecto, no una cotización vigente — antes de un uso productivo real conviene verificar el pricing actualizado en la documentación oficial de cada proveedor.

| Concepto | Estimación mensual (150 empleados, ~40 novedades/mes) |
|---|---|
| Llamadas al primer agente (Claude) | ~40 llamadas/mes |
| Llamadas al segundo agente (DeepSeek), asumiendo ~20% de casos requieren 1-2 reintentos | ~15-20 llamadas/mes |
| Costo relativo por llamada | Claude: mayor costo/token, pero bajo volumen. DeepSeek: menor costo/token, absorbe el volumen variable de reintentos sin escalar tanto el gasto total. |
| Ahorro estimado vs. usar el modelo más caro en ambos pasos | Al concentrar el modelo premium solo en la tarea que lo justifica (interpretación), y usar un modelo económico en la tarea de mayor frecuencia (validación con reintentos), el costo total del ciclo HITL se reduce significativamente frente a usar Claude en los dos pasos, sin pérdida de calidad en la validación mecánica. |

### 6.3 Otras medidas de optimización de costos

- **Max Tokens acotado** en ambos agentes, ya que las respuestas son JSON estructurado de tamaño predecible, no texto abierto.
- **Catálogo pasado como texto plano resumido** (no JSON anidado) al prompt, reduciendo tokens de entrada en cada llamada.
- **Corte de reintentos** (ver sección 7) evita loops que multiplicarían el gasto en llamadas indefinidamente.

---

## 7. Entregable 4 — Seguridad y resiliencia (20%)

### 7.1 Minimización de datos

- El agente de IA nunca recibe columnas de sueldos ya calculados (Neto, Bruto, Aportes) — solo datos estructurales y de novedades. Esto limita la exposición de información sensible del empleado a lo estrictamente necesario para la tarea.
- Los datos de prueba usados durante el desarrollo (nombres, legajos) están anonimizados antes de subirse como evidencia a este repositorio.

### 7.2 Rutas de error implementadas

| Situación | Manejo |
|---|---|
| Concepto de novedad no reconocido en el catálogo | No se autoprocesa; cae a revisión humana con motivo explícito |
| Legajo no encontrado (posible alta) | No se crea un empleado nuevo sin confirmación; va a HITL pidiendo los datos estructurales obligatorios |
| Dato incompleto (ej. licencia sin fecha de fin fuera de la regla de ART) | Se marca como incompleto, no se asume el valor |
| Formato de fecha o texto no estándar en la respuesta manual | El segundo agente normaliza formatos razonables (fechas, mayúsculas/minúsculas) antes de rechazar |
| Falla de parseo de la respuesta de la IA | Manejo explícito de error en el código, no se asume estructura válida por defecto |
| Reintentos indefinidos en el loop HITL | Ver punto 7.3 |

### 7.3 Filtro anti-bucle infinito

El ciclo de revisión humana (Loop Over Items) procesa un caso a la vez, y solo devuelve el control al loop principal cuando el caso queda resuelto (estado "ok"); un caso no resuelto vuelve únicamente al paso de reenvío de mensaje, sin re-ingresar como ítem nuevo del loop — evitando la multiplicación de tareas pendientes.

Agregamos un contador explícito de intentos por caso (ej. máximo 3 reintentos), que corte el ciclo y derive el caso a revisión manual fuera del sistema si se supera el límite — mitigación adicional más allá del comportamiento actual.

### 7.4 Puntos de Human-in-the-loop (HITL)

1. **Despidos** (con o sin causa): nunca se procesan automáticamente, sin importar qué tan completa esté la información.
2. **Altas de empleados nuevos**: requieren confirmación humana de los datos estructurales obligatorios (Nombre, Categoría, Jornada, Fecha de ingreso, Situación contributiva) antes de crear el registro.
3. **Datos incompletos o ambiguos**: cualquier campo que la IA no pueda determinar con certeza queda pendiente de aprobación, nunca se completa con un valor supuesto.

---

## 8. Entregable 5 — Dashboard de control (20%)

🔗 **[link/ Shared View con KPIs]**
---

## 9. Test de estrés y camino infeliz

Casos probados durante el desarrollo:

- ✅ Novedad simple (vacaciones) → procesamiento automático correcto, incluyendo recálculo de Días Trabajados.
- ✅ Despido → correctamente derivado a revisión humana, sin autoprocesarse.
- ✅ Alta de empleado nuevo con datos incompletos → sistema pide específicamente los campos faltantes, no crea la fila hasta tenerlos todos.
- ✅ Alta de empleado nuevo con datos completos → crea la fila nueva (Append) en Google Sheets.
- ✅ Corrección manual con formato de fecha no estándar (dd/mm/aaaa en vez de dd-mm-aaaa) → normalizado automáticamente por el segundo agente.
- ✅ Corrección manual con opción de texto en minúsculas ("normal" en vez de "NORMAL") → normalizado automáticamente.
- ⚠️ Se detectó y corrigió durante el desarrollo un bug de multiplicación de items en el loop HITL (documentado como aprendizaje del proyecto, ver ANEXOS.md).

📸  [Ver ejemplos de distintos casos ejecutados](ANEXOS/Ejecuciones)

---

## 10. Trabajo Futuro / próximos pasos

**Sub-workflow de mantenimiento de Google Sheets**: un flujo separado para (a) limpiar/resetear las columnas de días de la hoja "Empleados" al inicio de cada nuevo período de liquidación, y (b) actualizar la hoja "Escala/Convenio" cuando cambie el acuerdo paritario, sin tener que hacerlo a mano. Quedó fuera de esta entrega por alcance y tiempo, pensado como la siguiente iteración del proyecto.

---

## 11. JSON del flujo de n8n

📋  [Ver JSON completo del workflow](ANEXOS/Json%20completo%20n8n.txt)

---

## 12. Video demo

🎥

---

[Ver Anexos](ANEXOS) para el detalle completo de todos los prompts de IA y JSON de ejemplo utilizados durante el desarrollo.
