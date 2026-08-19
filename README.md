# Sistema de Triage de Consultas con IA

**Entrega final — Curso de Automatización con IA**
Autor: Guido Sánchez

Automatización del triage de consultas que empresas envían por correo a la Secretaría de Industria y Comercio. Un correo entra, un modelo de lenguaje lo clasifica y redacta un borrador de respuesta consultando una base de conocimiento interna, y el borrador queda a la espera de aprobación humana. **Ninguna respuesta se envía al remitente sin validación previa de una persona.**

> **Todos los datos son ficticios.** Los regímenes, reglamentos, empresas, correos y normativa citada fueron generados específicamente para este trabajo. Ninguna consulta real de la Secretaría ingresa al sistema.

---

## Enlaces de la entrega

| Recurso | Enlace |
|---|---|
| **Video demo** (6 min) | https://youtu.be/_abwoL8vkRY |
| **Dashboard de control** (KPI y tasa de errores) | https://roasted-hiss-be6.notion.site/Dashboard-de-Control-3c1a0733e0ae807ba9d4c033112a5dac |
| **Espacio del proyecto** (contiene las dos bases y el dashboard) | https://roasted-hiss-be6.notion.site/Triage-Consultas-Proyecto-Final-3bfa0733e0ae80a0b47acbe2d4c928e1 |
| Base de datos — Consultas (registro) | https://roasted-hiss-be6.notion.site/3bfa0733e0ae80068cd8e562250b48f1 |
| Base de datos — Base de conocimiento | https://roasted-hiss-be6.notion.site/3bfa0733e0ae80c296e8f600519a9861 |

Todos los enlaces de Notion están publicados en modo lectura.

**Índice del video:** `0:00` arquitectura del sistema · `3:00` aprobación humana y envío · `5:10` dashboard de control

---

## Archivos del repositorio

| Archivo | Contenido |
|---|---|
| `Documentacion-tecnica.pdf` | Diagrama de arquitectura, estructuras de datos, matriz de costos y documentación de seguridad y resiliencia |
| `Proyecto_final_-_Triage_Consultas.json` | Workflow completo de n8n, listo para importar |

La evidencia del flujo en ejecución —correo entrante, procesamiento, registro en Notion, notificación por Slack, aprobación y respuesta final— se muestra en el video demo.

---

## Stack

| Categoría | Herramienta |
|---|---|
| Orquestador | n8n (autoalojado) |
| Base de datos | Notion — dos tablas relacionadas |
| Procesamiento IA | Cohere (Command), vía Basic LLM Chain |
| Canal de entrada | Gmail Trigger |
| Canal de salida | Slack (notificación) y Gmail (respuesta) |
| Dashboard | Notion, vista publicada |

---

## Arquitectura

Un único workflow con **dos ramas que no están conectadas entre sí**. Ambas se comunican exclusivamente a través del estado persistido en Notion.

**Rama 1 — Ingesta y procesamiento**

```
Gmail Trigger → Set (normaliza) → Notion (lee base de conocimiento)
→ Limit → Basic LLM Chain + Cohere → Notion (crea registro) → Slack
```

**Rama 2 — Aprobación y envío**

```
[humano cambia el estado en Notion]
→ Notion Trigger → Notion (lee la fila) → If (doble condición)
→ Gmail (envía) → Notion (marca Finalizado)
```

**Ruta de error:** las salidas de fallo del nodo de IA y del nodo de escritura convergen en un registro de error en Notion, que alimenta la tasa de errores del dashboard.

n8n no mantiene estado entre ejecuciones, de modo que un flujo no puede quedarse esperando una aprobación humana. Por eso está partido en dos: la rama 1 procesa y persiste; la rama 2 se dispara de forma independiente cuando el estado cambia.

---

## Los tres caminos de resolución

El modelo no solo decide qué responder: decide también **si puede responder**.

| Tipo de resolución | Cuándo se aplica | Borrador generado |
|---|---|---|
| **Responde IA** | Existe una ficha aplicable y no está marcada para revisión | Respuesta completa, con cita de la ficha y la normativa |
| **Requiere humano** | No hay ficha aplicable, la ficha está marcada, se pide una excepción o se menciona un expediente en curso | Acuse de recibo, sin contenido normativo |
| **Solicita aclaración** | La consulta no aporta información suficiente para identificar el tema | Pedido cordial de precisión |

En los tres casos el borrador queda sujeto a aprobación humana. La distinción no define *si* se revisa, sino *qué tipo* de intervención se requiere.

El sistema está deliberadamente sesgado a derivar: el costo de derivar de más es que un agente lea un correo; el de responder de menos, que la Secretaría emita información incorrecta con carácter oficial.

---

## Check de seguridad de la consigna

**Filtro contra bucles infinitos.** La rama 2 escribe en la misma base que dispara su propio disparador. El nodo `Filtra aprobadas sin responder` exige dos condiciones simultáneas:

```
Estado == "Aprobado por Humano"   AND   Respuesta enviada == false
```

Al enviarse la respuesta se marca la casilla, de modo que la siguiente activación del disparador ya no cumple el filtro. Se produce una ejecución adicional, no un bucle. El mismo mecanismo evita el reenvío duplicado si el correo sale pero la escritura posterior en Notion falla.

**Tipos de dato en los filtros.** La primera condición compara cadenas de texto; la segunda, un valor booleano. Ambas están declaradas con el tipo que corresponde, bajo validación estricta.

**Prompt dinámico.** El prompt se construye en tiempo de ejecución con los datos del correo entrante y con las fichas leídas de Notion. No hay contenido fijo en el código: agregar o modificar una ficha en la base cambia el comportamiento del sistema sin tocar el workflow.

---

## Cómo importar el workflow

1. En n8n: **Workflows → Import from File** y seleccionar el `.json`.
2. Reconfigurar las cuatro credenciales (Gmail, Notion, Cohere, Slack). El archivo exportado **no contiene claves**: solo referencias internas al almacén de credenciales de n8n.
3. Apuntar los nodos de Notion a las bases propias.
4. Configurar el filtro por remitente en el Gmail Trigger.
