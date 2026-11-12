# PARES: pasos del diagrama funcional

Los pasos se agrupan por flujo, cada uno con su color y su letra: I (ingesta), H (buscador de hechos), P (prompt libre), S (salida en PDF) y A (administración). Las flechas grises sin número son de autenticación, de acceso a datos y de seguimiento.

## I · Ingesta y tratamiento de documentos

| Paso | Detalle |
|---|---|
| I1 | El administrador entra en la vista de administración del frontal web, autenticado con su cuenta de Google o Microsoft y con el rol Administradores. |
| I2 | Sube los escaneos a través de la API, que comprueba el token y el rol. |
| I3 | La API guarda cada escaneo en el almacén de originales. El máster no se modifica nunca. |
| I4 | La llegada de un escaneo nuevo pone en marcha el orquestador de ingesta. Desde aquí, el orquestador registra en el seguimiento por página cada paso: fase, número de intento, puntuación y motivo del juez, y versión de imagen usada. |
| I5 | Pre-procesado básico de la imagen: enderezar, quitar ruido y ajustar el contraste. |
| I6 | Detección de layout: bloques de texto, tablas y figuras, con sus coordenadas para recortar las imágenes. |
| I7 | El LLM multimodal transcribe el manuscrito, lo corrige por contexto y describe las imágenes (mapas, sellos, firmas). |
| I8 | Un LLM distinto actúa como juez de calidad (LLM as a judge). Devuelve una puntuación por bloque, el motivo (tinta desvaída, manchas, baja resolución, letra ilegible…) y las zonas afectadas. El orquestador decide de forma determinista comparando la puntuación con el umbral y mirando el número de intento. |
| I9 | Si la puntuación queda por debajo del umbral en el primer intento, la página pasa a restauración avanzada. El motivo del juez elige el modelo especializado: binarización y realce, limpieza de manchas o superresolución. La imagen restaurada se guarda como versión nueva; el original no se toca. |
| I10 | Se repite la transcripción con la imagen restaurada y el juez vuelve a puntuar. Solo hay un reintento. |
| I11 | Si la puntuación sigue por debajo del umbral tras el reintento, la página pasa a revisión humana con ambas transcripciones, el motivo del juez y las zonas marcadas. |
| I12 | El revisor (paleógrafo) entra en el portal de revisión. |
| I13 | Corrige la transcripción en la tarea de revisión. |
| I14 | Se guardan la transcripción validada y la corrección. Las correcciones forman el conjunto de referencia. |
| I15 | La página validada, por el juez (puntuación ≥ umbral) o por el revisor, pasa al enriquecimiento. |
| I16 | Un LLM en modo JSON con esquema extrae fechas, hechos, entidades y relaciones, y las enlaza con el catálogo de hechos conocidos o crea hechos identificados pendientes de validar. Se guardan en el grafo con la referencia al escaneo. |
| I17 | El texto enriquecido pasa a generar embeddings. |
| I18 | Se indexan en el índice híbrido (texto y vectores) con filtros por fecha, hecho y estado de validación. |

## H · Buscador de hechos

| Paso | Detalle |
|---|---|
| H1 | El usuario explora los hechos en la vista Usuario: filtra por fecha, tipo o entidad, y elige uno. |
| H2 | La petición llega a la API (síncrona). |
| H3 | La API la pasa al buscador de hechos, que es una cadena fija. |
| H4 | Si hay un resumen vigente en la caché, se usa ese. |
| H5 | Si no, la capa de acceso a datos recupera del grafo el hecho y su subgrafo, y del índice los pasajes relacionados. Solo devuelve datos validados. |
| H6 | El LLM de resumen genera el resumen con citas a los escaneos, y el resumen se guarda en la caché. |
| H7 | El buscador devuelve la respuesta a la API. |
| H8 | La vista Usuario muestra el resumen y los enlaces a los escaneos. |

## P · Prompt libre

| Paso | Detalle |
|---|---|
| P1 | El usuario escribe su petición en lenguaje natural y elige el formato: texto o PDF. |
| P2 | La petición llega a la API asíncrona, que devuelve el progreso mientras el agente trabaja. |
| P3 | Los guardarraíles de entrada filtran la *prompt injection*, las consultas fuera de tema y el contenido ofensivo. |
| P4 | La petición pasa al agente ReAct, que tiene un máximo de iteraciones. |
| P5 | El agente usa las herramientas del servidor MCP: buscar hechos, consultar el grafo, buscar pasajes y obtener el escaneo. El MCP fija el máximo de iteraciones por sesión, aplica la identidad del usuario y deja traza de cada llamada. |
| P6 | El MCP consulta la capa de acceso a datos, que solo devuelve datos validados. |
| P7 | El juez de respuestas comprueba que cada dato procede de fuentes validadas. |
| P8 | Si falta respaldo, el agente tiene un reintento sobre los campos sin respaldo, que cuenta para el límite del MCP. Si sigue sin respaldo, el campo sale como «sin datos» y la respuesta se marca como parcial. Cada veredicto e intento queda en la auditoría. |
| P9 | Los guardarraíles de salida bloquean fugas y contenido no respaldado. |
| P10 | La respuesta, en JSON estructurado, vuelve a la API. |
| P11 | La vista Usuario muestra la respuesta como texto. |

## S · Salida en PDF

| Paso | Detalle |
|---|---|
| S1 | Si el usuario eligió PDF, la API pide al generador que convierta la respuesta JSON en un PDF y devuelve un enlace de descarga temporal. Vale para el buscador de hechos y para el prompt libre. |

## A · Administración y control del modelo

| Paso | Detalle |
|---|---|
| A1 | El administrador entra en la vista de administración. |
| A2 | Sus operaciones pasan por la API, que comprueba el rol Administradores. |
| A3 | La API las envía al backend de administración. |
| A4 | Gestión de usuarios y asignación de roles. |
| A5 | Validación o descarte de los hechos identificados, y mantenimiento del catálogo de hechos conocidos. Lo que se aprueba pasa a ser visible para los usuarios. |
| A6 | Calibración del modelo: el conjunto de evaluación pasa por el transcriptor y el juez, y se miden la tasa de error y el acuerdo entre el juez y los revisores. El administrador ajusta los umbrales, los reintentos y los límites del MCP. |
| A7 | Consulta de las trazas (decisiones del juez, revisiones, llamadas a herramientas) y de los informes de uso, como las respuestas parciales o sin datos que indican qué validar primero. |

## Capas transversales

Afectan a todos los componentes, así que no llevan número:
- **Auditoría de decisiones:** juez, revisiones, admins y herramientas del agente.
- **Logs, métricas y alarmas:** técnicas y funcionales.
- **Autenticación y autorización:** roles y mínimo privilegio.
- **Protección:** límite de peticiones, cifrado y secretos.
- **Copias de seguridad y réplica DR.**
