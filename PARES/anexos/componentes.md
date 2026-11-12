# PARES: componentes técnicos (AWS)

Detalle de los componentes del diagrama técnico y del uso que tiene cada uno en la solución. Las Lambdas se agrupan por función (el enunciado permite agrupar funciones en un único icono).

## 1. Capa de acceso

| Componente | Uso en la solución |
|---|---|
| Amazon CloudFront + Amazon S3 (front) | Sirve el frontal web (buscador de hechos, prompt libre y consola de administración). |
| AWS WAF | Límite de peticiones por IP y reglas contra ataques web comunes delante de CloudFront y API Gateway. |
| Amazon Cognito (User Pool) | Login federado con Google y Microsoft. Grupos `Administradores` y `Usuarios`. Emite los tokens JWT que usan la API y el MCP (*on-behalf-of*). |
| Amazon API Gateway (REST) | API del buscador de hechos, la administración y las descargas, con autorizador de Cognito y *throttling*. |
| Amazon API Gateway (WebSocket) | Consulta asíncrona con progreso para el prompt libre: el agente puede superar el tiempo máximo de una petición síncrona. |

## 2. Ingesta y tratamiento (IDP)

| Componente | Uso en la solución |
|---|---|
| Amazon S3 `escaneos-originales` | Másteres de alta resolución, inmutables (versionado + Object Lock). Clase según antigüedad (ver §7). Replicación entre regiones (CRR): RPO ≈ 0. |
| Amazon S3 `visualizacion` | Copias ligeras de cada página y recortes de imágenes para el visor, servidas por CloudFront. |
| Amazon EventBridge | Detecta escaneos nuevos y lanza el pipeline. También programa la calibración periódica. |
| AWS Step Functions (pipeline IDP) | Orquesta la ingesta: pre-proceso → layout → transcripción → juez → restauración y reintento → A2I → enriquecimiento → indexación. Toma la decisión determinista con el veredicto del juez: puntuación ≥ umbral, aceptada; < umbral en el primer intento, restauración y reintento; < umbral tras el reintento, A2I. Lleva el contador de intentos y gestiona la espera de la revisión humana. |
| Amazon DynamoDB (seguimiento por página) | Estado actual de cada página: fase, número de intento, puntuación y motivo del juez, modelo de restauración aplicado y versión de imagen usada. Lo escribe Step Functions en cada paso; permite saber dónde está cada página y reanudar. Las decisiones también van al registro de auditoría. |
| AWS Lambda (funciones de ingesta) | Pre-proceso básico de imagen (enderezar, ruido, contraste), llamadas a Textract y Bedrock, escritura en OpenSearch y Neptune. |
| Amazon SageMaker (inferencia asíncrona, restauración) | Restauración avanzada con modelos especializados de imagen documental, solo si el juez da puntuación baja. El motivo del juez elige el modelo: binarización y realce para tinta desvaída, limpieza de manchas y transparencias, o superresolución para baja resolución. GPU y escala a cero cuando no hay trabajo. La imagen restaurada se guarda como versión nueva; el original no se toca. Un único reintento. |
| Amazon Textract | Análisis de *layout*: bloques de texto, tablas y figuras, con sus coordenadas para recortar imágenes. |
| Amazon Bedrock: modelo multimodal | Transcribe el manuscrito, corrige por contexto y describe las imágenes (mapas, sellos, firmas). |
| Amazon Bedrock: modelo juez | LLM as a judge, distinto del transcriptor. Devuelve un veredicto JSON: puntuación por bloque, motivo (tinta desvaída, manchas, baja resolución, letra ilegible…) y zonas afectadas. No decide: la decisión la toma Step Functions con el umbral. |
| Amazon Augmented AI (A2I) | Revisión humana de las páginas que siguen por debajo del umbral tras el reintento, por paleógrafos (personal privado). La tarea muestra ambas transcripciones, el motivo del juez y las zonas marcadas. Sus correcciones alimentan el conjunto de referencia. |
| Amazon Bedrock: enriquecimiento | En modo JSON con esquema: fechas, hechos, entidades (naves, personas, tercios, lugares) y relaciones, con referencia al escaneo. Normaliza las entidades. |
| Amazon Bedrock: embeddings | Vectoriza el texto enriquecido para la búsqueda híbrida. |
| Amazon S3 `procesados` | Transcripciones, JSON de metadatos, recortes de imágenes y conjunto de referencia (entrenamiento y evaluación separados y versionados). |

## 3. Conocimiento

| Componente | Uso en la solución |
|---|---|
| Amazon OpenSearch Service | Índice híbrido (BM25 + vectorial) con filtros por fecha, hecho y estado de validación. Multi-AZ. Reconstruible desde S3. |
| Amazon Neptune | Grafo de hechos y entidades: catálogo de hechos conocidos, hechos identificados y su estado de validación, todo con referencia al escaneo. Multi-AZ. |

## 4. Consulta

| Componente | Uso en la solución |
|---|---|
| AWS Lambda (buscador de hechos) | Cadena secuencial: filtra los hechos en Neptune, recupera los pasajes de OpenSearch y pide el resumen a Bedrock. Solo usa datos validados. |
| Amazon DynamoDB (caché) | Caché de resúmenes por hecho. Se invalida cuando entra un documento que afecta a ese hecho. |
| Amazon ECS Fargate (agente) | Agente ReAct (LangChain) del prompt libre, con `max_iterations` como segunda barrera. |
| Amazon ECS Fargate (servidor MCP) | Expone las herramientas del agente: buscar hechos, consultar el grafo, buscar pasajes y obtener el escaneo. Fija el máximo de iteraciones por sesión, aplica la identidad del usuario y registra cada llamada. |
| Amazon Bedrock Guardrails | Entrada: *prompt injection*, consultas fuera de tema y contenido ofensivo. Salida: comprobación de que la respuesta está respaldada por el contexto. |
| Amazon Bedrock: juez de respuestas | Comprueba que cada dato de la respuesta procede de fuentes validadas. Si no, el agente tiene un reintento (número configurable, cuenta para el límite del MCP); si sigue sin respaldo, el campo sale como «sin datos» y la respuesta se marca como parcial. |
| AWS Lambda (generador de PDF) | Convierte la respuesta JSON en PDF cuando el usuario lo pide. |
| Amazon S3 `resultados` | PDF generados, con caducidad automática. Se descargan con una URL prefirmada temporal. |

## 5. Administración y control del modelo

| Componente | Uso en la solución |
|---|---|
| AWS Lambda (administración) | Subida de documentos (URL prefirmada), gestión de usuarios y grupos en Cognito, y validación o descarte de hechos identificados. |
| Calibración (Step Functions programado) | Pasa el conjunto de evaluación por el transcriptor y el juez. Mide la tasa de error y el acuerdo entre el juez y los humanos. |
| AWS Systems Manager Parameter Store | Umbrales del juez, número de reintentos y límites del MCP, ajustables por el admin y versionados. |
| Amazon Athena + Amazon QuickSight | Informes de uso y de calidad: consultas, respuestas parciales, cola de A2I y coste por consulta. |

## 6. Capas transversales

En el diagrama técnico, las etiquetas de color sobre cada tarjeta marcan qué componentes usan una capa específica (auditoría, copia DR, WAF, secretos). La línea gris punteada recoge la telemetría de todos los componentes.

| Componente | Uso en la solución | Se conecta con |
|---|---|---|
| Amazon DynamoDB (auditoría) | Registro de decisiones: veredictos del juez, intentos y restauraciones, revisiones A2I, aprobaciones de admins, llamadas a herramientas del agente y veredictos del juez de respuestas. | Lo escriben Step Functions, el juez de calidad, A2I, el servidor MCP, el juez de respuestas, el backend de administración y la calibración. Lo leen Athena y QuickSight para trazas e informes. |
| Amazon CloudWatch (Logs, Metrics, Alarms) | Logs, métricas técnicas y funcionales, y alarmas. | Todos los componentes envían logs y métricas. Las alarmas van a SNS. Athena y QuickSight leen los logs exportados. |
| AWS X-Ray | Trazas distribuidas de cada petición de extremo a extremo. | API Gateway, Lambdas, Step Functions, agente y MCP. |
| Registro de invocaciones de Bedrock | Log de cada inferencia (prompt, respuesta y modelo) para auditoría y control de deriva. | Todos los usos de Bedrock; se guarda en CloudWatch y S3. |
| AWS CloudTrail | Auditoría de las acciones sobre la cuenta y los servicios. | Toda llamada a la API de AWS, sin integración por componente. |
| Amazon SNS | Avisos a los admins: respuestas no respaldadas, cola de A2I creciendo, deriva de la calibración. | Recibe las alarmas de CloudWatch y avisa por correo. |
| AWS IAM | Roles de mínimo privilegio por componente. | Cada Lambda, tarea de Fargate, máquina de Step Functions y endpoint de SageMaker tiene su propio rol. |
| AWS KMS | Cifrado en reposo. | Todos los almacenes: S3, OpenSearch, Neptune y DynamoDB. |
| AWS Secrets Manager | Secretos de cliente OIDC de Google y Microsoft, con rotación, y credenciales internas. | Cognito (federación con el IdP). |
| AWS WAF | Filtro de peticiones maliciosas y límite de peticiones. | CloudFront (consola y vista Usuario) y API Gateway. |
| AWS Backup + S3 CRR | DR por backup y restauración en otra región de la UE. | Réplica continua de escaneos y validaciones (S3); copias diarias de Neptune, OpenSearch y DynamoDB (seguimiento y auditoría). |
| Inferencia entre regiones de Bedrock | Disponibilidad del modelo si cae en una región. | Todos los usos de Bedrock. |

## 7. Almacenamiento por tipo de dato (temporalidad y criticidad)

| Dato | Acceso | Criticidad | Almacenamiento | Ciclo de vida y DR |
|---|---|---|---|---|
| Másteres de los escaneos | Intenso el primer mes (ingesta, restauración, A2I); después ocasional (reprocesos) | Máxima, irrecuperable | S3 Standard + versionado + Object Lock | A Glacier Instant Retrieval a los 30 días. Réplica DR continua en Glacier Deep Archive (copia de preservación) |
| Copias de visualización y recortes | Inmediato, con un patrón de uso impredecible | Media, regenerable desde el máster | S3 Intelligent-Tiering + CloudFront | Réplica continua (es lo que sirve el visor en DR) |
| Transcripciones y JSON de metadatos | En reindexaciones y reprocesos | Alta: base para reconstruir el índice y el grafo | S3 Standard (ficheros pequeños) | Réplica continua: replicar texto es más barato que volver a pagar la inferencia |
| Validaciones humanas (A2I y admins) y conjunto de referencia | Periódico (calibración, ajuste fino) | Máxima, irrecuperable | S3 Standard + versionado + Object Lock | Réplica continua (RPO ≈ 0) |
| Índice de búsqueda | Inmediato (milisegundos) | Media, reconstruible desde S3 | OpenSearch Service (Multi-AZ) | Snapshot diario a S3 replicado |
| Grafo de hechos | Inmediato (milisegundos) | Media: reconstruible; las validaciones están en S3 | Neptune (Multi-AZ) | AWS Backup diario con copia a otra región |
| Caché de resúmenes y sesiones del MCP | Inmediato | Baja, efímera | DynamoDB | TTL |
| Registro de auditoría | Inmediato los primeros 90 días; después, consulta puntual | Alta | DynamoDB (PITR) | Exportación a S3 Glacier Instant Retrieval a los 90 días |
| Logs técnicos | Inmediato los primeros 30 días; ante incidencias graves, en minutos | Media | CloudWatch Logs 30 días → S3 Standard | A Glacier Instant Retrieval a los 90 días (acceso en milisegundos para incidencias) |
| Logs de uso (informes) | Consultas analíticas, sobre todo recientes | Media | S3 Standard + Athena | Standard-IA a los 90 días |
| PDF generados | Solo descarga | Baja, temporal | S3 Standard | Expiran a las 24 h. El enlace prefirmado caduca a los 15 min |

Supuesto: el enunciado no fija la retención total de la auditoría ni de los logs; queda «a definir por el Ministerio».
