# Agente de Becas y Movilidad — Equipo 5

Agente que arma el expediente de una postulación (beca deportiva o movilidad Criscos) y lo deja listo para que una persona decida. **Nunca aprueba ni rechaza.**

El detalle técnico completo está en el documento fuente: [`Spec del agente de becas y movilidad- Equipo5 (limpio).docx`](./Spec%20del%20agente%20de%20becas%20y%20movilidad-%20Equipo5%20%28limpio%29.docx).

## Qué hace

Detecta documentos faltantes e inconsistencias, consolida las fuentes y deja un checklist donde cada dato dice de dónde salió.

- **Problema que resuelve.** Adelanta la revisión de completitud de cada expediente y deja el rastro de cada dato, de cada paso de los agentes y de cada acción humana.
- **Quién lo usa.** Admisiones opera los expedientes y las autoridades académicas deciden.
- **Alcance del MVP.** Beca deportiva y movilidad Criscos. Prácticas se agrega después con un perfil nuevo.
- **Qué queda fuera.** La etapa posterior a la adjudicación (pasajes, seguros, matrícula) y verificar si un documento es auténtico.

## Reglas que no se negocian

- La decisión es humana. El sistema solo dice "completo", "incompleto" o "con inconsistencias".
- Los agentes razonan y el código garantiza. La evidencia, el estado y los límites los imponen componentes deterministas.
- Ningún veredicto sin evidencia verificable en el log.
- Un fallo nunca equivale a "cumple". Queda como "no verificable".
- Todo dato dice de dónde salió: fuente, ubicación, método y confianza.
- Mínimo privilegio. Cada agente usa solo las herramientas que necesita, todas de solo lectura.
- Los criterios salen del perfil. Ningún agente define requisitos, criterios, fechas de referencia ni umbrales; todo eso lo fija el perfil de la convocatoria (D3).

## Cómo funciona

El código reparte el trabajo según el perfil, los workers devuelven datos y veredictos con evidencia, el coordinador revisa esos resultados y devuelve lo insuficiente, y el código lo valida todo antes de que una persona decida. Nada llega a la persona sin pasar por la puerta de evidencia.

### Un expediente, paso a paso

1. **Ingesta.** El estudiante entrega sus documentos por la plataforma, el correo o la ventanilla. La ingesta valida cada archivo y lo guarda por hash.
2. **Plan de trabajo.** El código lee el perfil de la convocatoria y arma el plan con los requisitos por verificar. El coordinador lo recibe, pero no puede agregar ni quitar requisitos.
3. **Extracción en paralelo.** El motor reparte las tareas del plan. El Worker de documentos lee y clasifica los archivos; el Worker de registros consulta matrícula, Cartera, finanzas y convenios. El coordinador revisa que cada resultado tenga la forma y la evidencia pedidas, y puede devolver una tarea hasta dos veces; después, lo que falte queda "no verificable".
4. **Consolidación.** Cada dato entra al expediente con su fuente, ubicación, método y confianza. El Consolidador une las fuentes, señala duplicados y contradicciones, y cruza la cédula y el nombre de cada documento con la matrícula.
5. **Verificación.** El motor pide al Worker verificador un veredicto por requisito. Todo cálculo (promedio, fechas, vigencias) sale de una herramienta y queda en el log.
6. **Puerta de evidencia.** Comprueba que cada veredicto tenga evidencia real, revisa que no falte ningún requisito y calcula el estado del expediente. Si falta evidencia, no cita el cálculo registrado o se apoya en un dato con confianza bajo el umbral, el requisito baja a "no verificable". Los requisitos condicionales que no aplican quedan "no aplica".
7. **Subsanación.** Si el expediente queda incompleto, el Worker de subsanación redacta el borrador de la solicitud de faltantes. El código comprueba que liste exactamente los requisitos pendientes, y una persona autoriza el envío.
8. **Revisión humana.** El operador abre el expediente, conversa con el coordinador, corrige o verifica lo que haga falta y lo pasa a la autoridad.
9. **Decisión.** La autoridad registra su decisión. El sistema la guarda y cierra el expediente.

Si llega un documento nuevo o una corrección, se crea una versión nueva del expediente y el ciclo se repite desde el paso 3. Los veredictos de los requisitos afectados se invalidan, y las verificaciones humanas se conservan para los datos que no cambiaron.

## Diagrama de flujo de datos (DFD)

- **Nivel 0 — Contexto.** El agente es una sola caja: recibe documentos, datos de sistemas, perfiles y acciones de personas, y entrega el expediente con su checklist, su evidencia y su estado.
- **Nivel 1 — Procesos y almacenes.** 9 procesos sobre 3 almacenes:
  - **D1 · Archivos.** Documentos originales, guardados por hash y sin modificarse.
  - **D2 · Log de eventos.** La fuente de verdad: hechos, tareas, veredictos, decisiones y conversaciones. Es el bus del sistema.
  - **D3 · Perfiles.** Requisitos y criterios de aceptación de cada convocatoria.

| # | Proceso | Lo ejecuta | Lee | Escribe |
|---|---|---|---|---|
| 1.0 | Recibir archivos | Ingesta (código) | Entregas del estudiante y constancia de Bienestar | Archivos (D1) y eventos de entrega |
| 2.0 | Extraer datos | Worker de documentos (agente) | Archivos (D1) y la tarea | Datos con procedencia y confianza |
| 3.0 | Consultar sistemas | Worker de registros (agente) | La tarea | Datos de matrícula, Cartera, finanzas, convenios y convocatoria |
| 4.0 | Consolidar expediente | Consolidador (código) | Datos y correcciones | Expediente versionado, duplicados y conflictos |
| 5.0 | Coordinar el trabajo | Coordinador (agente) y motor (código) | Perfil (D3), expediente y mensajes | Plan de trabajo, revisión de resultados, tareas devueltas y respuestas |
| 6.0 | Verificar requisitos | Worker verificador (agente) | La tarea y el expediente | Veredictos con evidencia |
| 7.0 | Puerta de evidencia | Código | Veredictos y expediente | Requisitos evaluados y estado del expediente |
| 8.0 | Revisión humana | Operador, autoridad y servicio de revisión | Expediente, checklist y evidencia | Correcciones, verificaciones y decisión |
| 9.0 | Pedir faltantes | Worker de subsanación (agente), una persona y el notificador | Lista de faltantes | Borrador, autorización y aviso al estudiante |

## Los agentes

Un coordinador habla con las personas y supervisa a cuatro workers, que no hablan con nadie ni delegan. Todos los workers devuelven el mismo resultado: estado, qué hicieron, artefactos, evidencia, riesgos y una pregunta si quedaron bloqueados.

| Agente | Qué hace | Modelo | No puede |
|---|---|---|---|
| **Coordinador** | Recibe el plan que arma el código desde el perfil, revisa los resultados de los workers y devuelve una tarea si falta evidencia. Es el único que habla con las personas. | Qwen3-8B con razonamiento | Aprobar o rechazar, modificar datos por chat, definir requisitos o criterios, crear agentes, devolver una tarea más de dos veces ni leer el texto crudo de los documentos |
| **Worker de documentos** | Clasifica cada archivo, compara su tipo con la casilla donde se subió, extrae los campos y detecta documentos casi idénticos. | Qwen3-8B directo | Consultar sistemas ni emitir veredictos. Es el único que lee el texto crudo |
| **Worker de registros** | Consulta matrícula, obligaciones económicas (Cartera), pagos (finanzas y SGA), convenios y la convocatoria. | Qwen3-8B directo | Leer documentos ni modificar ningún sistema |
| **Worker verificador** | Para cada requisito revisa si se cumplen los criterios de aceptación y da un veredicto citando la evidencia. | Qwen3-8B con razonamiento | Calcular sin una herramienta registrada en el log, leer documentos crudos, decidir si un requisito condicional aplica ni fijar el estado del expediente |
| **Worker de subsanación** | Redacta el borrador de la solicitud de faltantes al estudiante. No envía nada. | Qwen3-8B directo | Enviar nada — una persona revisa y autoriza el envío |

## Componentes de código (sin modelos)

Son los que dan las garantías deterministas:

| Componente | Qué hace |
|---|---|
| **Ingesta** | Recibe las entregas por plataforma, correo o ventanilla, valida el archivo (tipo real, tamaño, antivirus) y lo guarda por hash. La constancia de Bienestar entra por aquí. |
| **Consolidador** | Versiona el expediente, deduplica por hash, aplica correcciones humanas, señala datos contradictorios, cruza cédula/nombre con matrícula, elimina el campo reservado (AC) e invalida veredictos afectados por una versión nueva. |
| **Puerta de evidencia y estado** | Valida la evidencia de cada veredicto, garantiza que todo requisito tenga resultado y calcula el estado del expediente. |
| **Motor de agentes** | Arma el plan desde el perfil, reparte las tareas y ejecuta el ciclo de cada agente con sus límites de pasos, tiempo, presupuesto y reintentos, registrando cada paso. |
| **Servicio de revisión** | Ofrece la bandeja, el visor de documentos, la conversación con el coordinador, las correcciones, la verificación manual y el registro de la decisión. |
| **Gestor de perfiles** | Publica las versiones del perfil de cada convocatoria (requisitos, criterios, tipos de documento válidos, fecha de referencia, periodo, umbrales de confianza), revisadas por el responsable normativo. |
| **Notificador** | Envía avisos internos y mensajes al estudiante solo después de una acción humana. |
| **Proyecciones** | Construyen las vistas (bandeja, línea de tiempo, auditoría) a partir del log de eventos. |

## Qué datos maneja

**Qué entra:**
- Documentos del estudiante: cédula, papeleta de votación, certificado de notas, carta de aceptación, pasaporte y visa, recibos y facturas, formularios firmados y certificado del club.
- Documentos de otras áreas: constancia de Bienestar sobre sanciones disciplinarias.
- Consultas a sistemas: matrícula, Cartera, finanzas, SGA, convenios y convocatoria.
- Acciones de las personas: correcciones, verificaciones manuales y la decisión.

**Cada dato registrado incluye:**

| Campo | Qué dice |
|---|---|
| Nombre | Qué dato es, por ejemplo "promedio general" o "vigencia del pasaporte" |
| Valor | El valor normalizado y el original, tal como aparece en la fuente |
| Fuente y ubicación | Documento, página y región; o sistema y consulta |
| Método | Lectura directa, plantilla, OCR, modelo, consulta o ingreso humano |
| Confianza | De 0 a 1. El perfil fija el umbral por dato y por método; un dato extraído por modelo tiene un tope inicial de 0,8. Por debajo del umbral, el resultado es "no verificable" |
| Quién lo produjo | Agente, modelo y versión de sus instrucciones, o la persona |
| Momento | Cuándo se leyó o se consultó |

### Requisitos verificados — Movilidad Criscos

| # | Requisito | Evidencia |
|---|---|---|
| 1 | Matriculado como alumno regular | Sistema |
| 2 | Materias aprobadas hasta el 4.º semestre | Certificado de notas y malla |
| 3 | Promedio mínimo de la convocatoria | Certificado de notas y convocatoria |
| 4 | Sin sanciones disciplinarias | Constancia de Bienestar |
| 5 | Al día en obligaciones económicas y reglamentarias | Sistema (Cartera) |
| 6 | Convocatoria y tres opciones de universidad | Formulario |
| 7 | Formularios firmados por el coordinador académico | Documento |
| 8 | Convenio de cooperación con el destino | Catálogo |
| 9 | Carta de aceptación del destino | Documento externo |
| 10 | Suficiencia de idioma, si el destino la exige | Documento; "no aplica" si el destino no la exige |
| 11 | Pasaporte vigente y, de ser el caso, visa | Documento; la visa "no aplica" si el destino no la exige |
| 12 | Recibo de $40 "Postulación Intercambio" | Sistema o recibo |

### Requisitos verificados — Beca deportiva

| # | Requisito | Evidencia |
|---|---|---|
| 1 | Factura de $5 por compra de solicitud (SGA) | Documento o sistema |
| 2 | Cédula y papeleta de votación actualizada, en PDF a color | Documento; el color lo comprueba una herramienta |
| 3 | Factura legalizada de pago total o inicial del periodo. No valen depósitos ni transferencias | Documento o sistema |
| 4 | Certificado del Club Indoamérica del periodo, firmado y sellado | Documento |
| 5 | Prórroga de pagos aprobada por Cartera, o factura del pago total | Documento o sistema |
| 6 | Solicitud de beca firmada | Documento |

El campo reservado (AC) de la beca deportiva es de uso interno; el Consolidador lo elimina antes de que llegue a un agente. Un documento de otro tipo (por ejemplo un depósito en lugar de una factura) cuenta como evidencia faltante, no como "no cumple".

## Resultado que produce

Cada requisito recibe un veredicto y el expediente recibe un estado que calcula el código.

| Nivel | Valores | Quién lo produce |
|---|---|---|
| Requisito | Cumple, no cumple, no verificable, no aplica | El Worker verificador, validado por la puerta de evidencia; "no aplica" lo decide el código según las condiciones del perfil |
| Expediente | Completo, incompleto, con inconsistencias | Una función de código sobre los veredictos |

El estado mide **integridad y coherencia, no elegibilidad**. Un expediente puede estar "completo" y tener un requisito que "no cumple" por umbral; la interfaz muestra ambas cosas juntas. "No cumple" significa que la evidencia es válida y verificable pero no alcanza el criterio; si la evidencia falta o no es del tipo exigido, el requisito cuenta como incompleto.

El estado se calcula en este orden:
1. **Con inconsistencias** — si hay datos que se contradicen o evidencia que no corresponde al estudiante.
2. **Incompleto** — si falta evidencia, es inválida o está pendiente de verificación.
3. **Completo** — en los demás casos.

Un requisito "no verificable" cuenta como incompleto hasta que una persona lo verifique; en el checklist aparece como "pendiente de verificación humana". No hay puntaje ni recomendación. La bandeja se ordena por plazo y antigüedad.

## Tecnología y modelos

### Stack

| Capa | Elección |
|---|---|
| Bus de eventos | PostgreSQL como log inmutable y cola, detrás de una interfaz reemplazable |
| Backend | Python, FastAPI y Pydantic |
| Interfaz | React con TypeScript, visor de PDF y panel de conversación |
| Herramientas de los agentes | Servidores MCP de solo lectura: OCR, lector de PDF, cálculo y sistemas institucionales |
| OCR y PDF | Tesseract como base, pypdfium2 y pdfplumber |
| Perfiles de convocatoria | Archivos versionados con los requisitos y sus criterios de aceptación |
| Archivos | Almacenamiento compatible con S3, direccionado por hash |
| Observabilidad | OpenTelemetry con identificador de expediente y de tarea |
| Despliegue | Docker Compose |

### Modelos y servidor

- **Modelo.** Qwen3-8B cuantizado (AWQ), servido con vLLM en el servidor privado de la universidad: 4 GPU RTX 4000, 128 GB de RAM y 40 hilos de CPU.
- **Modos.** Razonamiento activado para el coordinador y el verificador; modo directo para los demás agentes.
- **Referencias.** DeepSeek solo para comparar, con datos anonimizados. Qwen3.5-4B para desarrollo en portátil.
- **Antes de usar datos reales.** El servidor necesita clave de API y TLS, y hay que confirmar que vLLM arranca con llamada a herramientas.

## Seguridad y datos

- Los datos personales se tratan bajo la LOPDP, y los modelos corren dentro de la universidad.
- Solo el Worker de documentos lee el texto crudo. El resto recibe datos estructurados.
- Los agentes usan herramientas de solo lectura. Un documento con instrucciones ocultas no puede cambiar un veredicto, porque todo veredicto necesita evidencia real, ni meter un dato inventado, porque el código comprueba que el valor aparezca en la página y la región citadas.
- La conversación con el coordinador no modifica datos ni registra decisiones. Eso se hace con acciones explícitas de la interfaz, con el usuario autenticado.
- El servidor de modelos exige clave de API y TLS antes de usar datos reales.
- Cada expediente se cifra con su propia clave, y cada acceso y cada paso de un agente quedan auditados.

## Cómo se prueba

- **Sin datos simulados.** Casos de frontera para el código y las herramientas, y material real anonimizado para evaluar a los agentes.
- **Métrica crítica: falsos "cumple".** Cuando el sistema dice que cumple y no era así. El objetivo es cero en los casos evaluados.
- **Otras métricas.** Veredictos correctos frente a los de una persona, veredictos degradados por la puerta de evidencia, estabilidad entre corridas, correcciones por revisor, y tiempo y gasto por expediente.
- **Piloto en modo sombra.** El agente corre junto al proceso de Admisiones, sin tomar ninguna decisión.

## Fases de construcción

Las etapas van en orden, y cada una termina cuando se cumple su condición de salida.

| Etapa | Qué se construye | Termina cuando |
|---|---|---|
| **E0 — Fundaciones** | Repositorio, bus y log, contratos de eventos y tareas, motor de agentes y servidor de modelos asegurado | Un agente de prueba completa una tarea registrada contra el servidor de modelos |
| **E1 — Perfil y estado (beca deportiva)** | Perfil deportivo, ingesta, herramientas de cálculo, puerta de evidencia y cálculo del estado | Los casos de frontera pasan sin falsos "cumple" |
| **E2 — Documentos y verificación** | OCR, lectura de PDF, Worker de documentos y Worker verificador | La evaluación con material real anonimizado cumple los criterios, y lo que no llega queda "no verificable" |
| **E3 — Coordinador y revisión** | Coordinador, bandeja, visor, conversación, correcciones, subsanación y registro de la decisión | Piloto en modo sombra con Admisiones |
| **E4 — Registros y movilidad** | Worker de registros con herramientas MCP y perfil de movilidad Criscos | Acceso confirmado, o requisitos "no verificable" por diseño |
| **E5 — Producción** | Seguridad, LOPDP, retención, respaldo y despliegue | Revisión del DPD firmada y prueba de restauración superada |
| **Después del MVP** | Perfil de prácticas y etapa posterior a la adjudicación | Sobre el mismo sistema |

## Pendientes por confirmar

- Modelo exacto de las GPU y opciones de arranque de vLLM para herramientas.
- Motor de agentes: el de PumaNet u otro. Recomendación: un motor propio en Python que reutilice los patrones de PumaNet (presupuesto estricto, delegar y devolver, espera conjunta de resultados, chequeos deterministas y paquete de evidencia).
- Marco de tratamiento de datos con el DPD, plazo de retención y SSO institucional.
- Dónde se registra la decisión final: en este sistema o en el oficial.
- Convocatoria vigente, manual de becas deportivas, formularios y correo de Bienestar.
- Material real anonimizado para evaluar a los agentes.
- Quién envía la constancia de Bienestar y en qué formato.
- Valores iniciales de los umbrales de confianza y de los presupuestos de los agentes, a validar con material real anonimizado.

## Dónde está el detalle

El [SDD completo](./Spec%20del%20agente%20de%20becas%20y%20movilidad-%20Equipo5%20%28limpio%29.docx) define el agente en nueve fases, más un anexo con 40 decisiones y los riesgos:

| Fase | Qué define |
|---|---|
| 0 | Contexto, alcance, principios y preguntas abiertas |
| 1 | Dominio y catálogo de eventos |
| 2 | Arquitectura de agentes y del bus |
| 3 | Stack, servidor de modelos y librerías |
| 4 | Modelo de datos, definición de agente y contrato de tareas |
| 5 | Perfiles, verificación y estado |
| 6 | Revisión humana y conversación con el coordinador |
| 7 | Seguridad, LOPDP y auditoría |
| 8 | Pruebas y plan de entrega |
