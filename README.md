# Brasa y Alma – Asistente inteligente con RAG en n8n

## 1. Descripción del proyecto

Este proyecto consiste en el desarrollo de un asistente inteligente para **Brasa y Alma**, un emprendimiento de elaboración y venta de viandas. El objetivo es responder consultas frecuentes de clientes utilizando información previamente organizada del negocio y derivar a una persona aquellas situaciones que requieren confirmación o intervención humana.

La solución se desarrolló en n8n y utiliza un enfoque RAG (*Retrieval-Augmented Generation*): antes de generar una respuesta, el asistente recupera información relevante desde una base de conocimiento almacenada en Supabase.

## 2. Caso de negocio y problema identificado

Brasa y Alma recibe consultas relacionadas con:

- Menú, viandas y productos disponibles.
- Precios e ingredientes.
- Horarios de atención.
- Entregas y formas de retiro.
- Pedidos que necesitan confirmación.

La atención manual de estas consultas demanda tiempo y puede generar demoras, especialmente cuando se repiten preguntas similares. El asistente busca automatizar las consultas informativas que pueden responderse con datos confiables, sin reemplazar la intervención humana cuando sea necesaria.

### Estimación del retorno de inversión (ROI)

Como hipótesis de trabajo se considera:

- 30 consultas por día.
- 3 minutos promedio de atención manual por consulta.
- 20 días de actividad al mes.

Esto representa aproximadamente **30 horas mensuales** destinadas a responder consultas. Si el asistente pudiera resolver automáticamente el 70 % de ellas, el ahorro potencial sería de **21 horas mensuales**.

Esta cifra es una estimación inicial y no un resultado medido en producción. El ahorro real dependerá del volumen de consultas, la calidad de la información disponible y el porcentaje de consultas que efectivamente pueda resolver el asistente. No se monetiza el ahorro porque no se definió un costo por hora de trabajo.

## 3. Arquitectura general

La solución está organizada en tres workflows de n8n:

1. **Indexación:** obtiene la información desde Google Sheets, prepara los textos y los almacena en Supabase Vector Store.
2. **Asistente IA:** recibe consultas mediante un Webhook, recupera información relevante con RAG y genera una respuesta o deriva la consulta a una persona.
3. **Gestión de errores:** captura errores de los workflows asociados y registra información del incidente en Google Sheets.

La arquitectura utiliza dos fuentes de entrada complementarias: Google Sheets para cargar y actualizar la información del negocio, y un Webhook para recibir las consultas de los clientes. Estas entradas se encuentran en workflows separados.

## 4. Tecnologías utilizadas y justificación

- **n8n:** plataforma de automatización que permite conectar las distintas fuentes, servicios y componentes de IA.
- **Google Sheets:** almacenamiento y administración de la información del negocio, las consultas pendientes y los errores.
- **Supabase con pgvector:** base de datos vectorial utilizada para almacenar los documentos y recuperar información por similitud semántica.
- **Google Gemini:** modelo de lenguaje utilizado para interpretar las consultas y redactar respuestas.
- **Gemini Embeddings:** modelo utilizado para generar los vectores que representan los documentos de la base de conocimiento.
- **Webhook:** punto de entrada para recibir las consultas.

### Elección de un modelo en la nube

Se eligió Google Gemini como modelo en la nube porque permite integrar un modelo de lenguaje y embeddings con los servicios utilizados en el proyecto, sin tener que administrar infraestructura local para ejecutarlos.

Esta decisión facilita el desarrollo y las pruebas dentro del plazo del trabajo práctico. Como contrapartida, existe dependencia del proveedor, de la conexión a internet y de los límites de uso o costos del servicio. Durante las pruebas se observaron respuestas de límite de cuota (errores 429) en el nivel gratuito de Gemini.

La elección de un modelo local, por ejemplo mediante Ollama, podría evaluarse en una implementación futura si cambian los requisitos de privacidad, costos o disponibilidad de infraestructura. No es un requisito de esta implementación.

## 5. Workflow de indexación

El workflow **Indexación** carga la información desde dos pestañas de Google Sheets:

- `Productos`: contiene los productos, precios, ingredientes, descripción y disponibilidad.
- `Informacion_negocio`: contiene información general, como días y horarios de atención, entregas, retiro y confirmación de pedidos.

Los datos se leen, se transforman en textos y se unifican mediante un nodo Merge. Luego se envían a Supabase Vector Store, donde quedan almacenados como documentos con sus respectivos embeddings.

La base de conocimiento utilizada para las pruebas contiene ocho registros: tres productos y cinco documentos con información general del negocio.

### Ejemplos de productos indexados

| Producto | Precio | Ingredientes |
|---|---:|---|
| Canelones de verdura | $8.000 | Acelga, ricota y salsa |
| Pastel de papas | $7.500 | Papa, carne, queso y condimentos |
| Albóndigas con puré | $8.500 | Carne, papa y salsa |

Los precios y datos anteriores corresponden a los valores de ejemplo utilizados durante las pruebas del trabajo práctico. Deben actualizarse en la fuente de datos cuando cambie la información comercial.

## 6. Workflow del asistente IA y funcionamiento RAG

El workflow **Brasa y Alma - Asistente IA** recibe solicitudes mediante un Webhook configurado para el método POST y el path `brasa-y-alma`.

El cuerpo de la solicitud contiene los datos del contacto y el mensaje recibido, por ejemplo:

```json
{
  "telefono": "2210000000",
  "mensaje": "¿Cuánto cuestan los canelones de verdura?"
}
```

El AI Agent utiliza Google Gemini como modelo de lenguaje, Simple Memory para conservar contexto de conversación durante las pruebas y Supabase Vector Store como herramienta de recuperación de documentos.

El proceso general es:

1. El Webhook recibe el mensaje.
2. El AI Agent interpreta qué necesita saber el cliente.
3. El componente RAG consulta Supabase y recupera los documentos relevantes.
4. El modelo genera una respuesta basada en la información recuperada y en las instrucciones del sistema.
5. Si la consulta requiere confirmación o intervención humana, el agente utiliza una herramienta de Google Sheets para registrar la consulta pendiente.

### Configuración de recuperación

La recuperación se realiza sobre la tabla `documents` de Supabase, utilizando la función `match_documents` y un límite de cinco documentos recuperados. Los embeddings se generan con el modelo `gemini-embedding-001`, configurado con la dimensión compatible con los vectores almacenados.

## 7. System Prompt y criterios del agente

El agente tiene instrucciones para actuar como asistente virtual de Brasa y Alma. Sus principales criterios son:

- Responder de manera clara, cordial y breve.
- Utilizar la información recuperada desde la base de conocimiento.
- No inventar precios, ingredientes, horarios, disponibilidad ni zonas de cobertura.
- No confirmar pedidos, pagos ni disponibilidad.
- Derivar a una persona las consultas que requieren confirmación, como pedidos, entregas en zonas no documentadas, reclamos o solicitudes especiales.
- Registrar las consultas que requieren intervención humana en la pestaña `Consultas_pendientes` de Google Sheets.
- Si no encuentra información suficiente, reconocerlo y solicitar intervención humana en lugar de improvisar una respuesta.
- No afirmar que un pedido fue confirmado ni que el equipo ya fue notificado si eso no ocurrió.

El System Prompt completo se encuentra configurado en el nodo AI Agent del workflow exportado.

## 8. Salida real y registro de consultas pendientes

Como salida utilizable, el asistente puede registrar consultas que requieren seguimiento humano en Google Sheets, mediante la herramienta `registrar_consulta_pendiente`.

La pestaña `Consultas_pendientes` permite guardar los datos de la consulta y mantener un registro para que el equipo pueda revisarla y responder según corresponda.

Durante las pruebas se verificó el registro de una solicitud de dos porciones de canelones de verdura para el día siguiente. El agente no confirmó el pedido: lo derivó para revisión humana y creó el registro correspondiente.

## 9. Gestión de errores y casos límite

Se desarrolló un workflow independiente denominado **Brasa y Alma - Gestión de errores**, que utiliza un nodo Error Trigger y registra los incidentes en la pestaña `Errores` de Google Sheets.

Los campos registrados son:

- `fecha`: momento en que ocurrió el error.
- `workflow`: workflow en el que se produjo.
- `nodo`: último nodo ejecutado.
- `mensaje_error`: descripción del error.
- `url_ejecucion`: enlace a la ejecución correspondiente.

El workflow de gestión de errores se asignó desde la configuración de los workflows principales.

### Prueba de gestión de errores

Se realizó una prueba controlada generando intencionalmente un error desde un workflow temporal. Se verificó que el workflow de gestión de errores capturara el incidente y completara los campos correspondientes en la hoja `Errores`. El workflow temporal utilizado para la prueba fue desactivado después de la verificación.

### Casos límite contemplados

- **Información no disponible:** el agente evita inventar datos y deriva la consulta.
- **Pedido que requiere confirmación:** se registra en `Consultas_pendientes` y no se confirma automáticamente.
- **Consulta sobre cobertura de entrega:** si la zona no está documentada, se solicita revisión humana.
- **Límite de cuota del proveedor:** durante las pruebas se observaron errores 429 de Gemini. Esto evidencia la dependencia de los límites del servicio externo y la necesidad de monitorear su disponibilidad y consumo.

## 10. Pruebas realizadas

Se realizaron pruebas funcionales para comprobar la recuperación de información, la generación de respuestas y la derivación de consultas:

1. **Consulta de precio e ingredientes:** ante la pregunta por los canelones de verdura, el asistente respondió el precio de $8.000 e indicó sus ingredientes.
2. **Consulta de opciones con carne:** recuperó y presentó el pastel de papas y las albóndigas con puré con sus respectivos precios.
3. **Consulta de cobertura:** ante una consulta por entregas en City Bell, no inventó una respuesta y derivó el caso.
4. **Registro de pedido:** ante una solicitud de dos porciones de canelones para el día siguiente, generó un registro pendiente de revisión humana.
5. **Memoria de conversación:** se verificó el contexto asociado a un teléfono y el aislamiento entre conversaciones de distintos teléfonos.
6. **Gestión de errores:** se provocó un error de prueba y se comprobó su registro en Google Sheets.

## 11. Limitaciones y mejoras futuras

La implementación corresponde a un prototipo funcional para el trabajo práctico. Antes de utilizarlo con clientes reales sería necesario:

- Integrar el Webhook con un canal real de mensajería, como WhatsApp Business Platform. En esta etapa, el Webhook se utiliza para simular la recepción de consultas; no existe una integración real con WhatsApp.
- Configurar controles de seguridad y autenticación para proteger el endpoint del Webhook.
- Definir políticas de privacidad y tratamiento de los datos personales de los clientes.
- Implementar un mecanismo de respuesta alternativa ante fallos del modelo o de los servicios externos.
- Revisar periódicamente la información de Google Sheets y volver a ejecutar la indexación cuando existan cambios.
- Monitorear el consumo, los límites de cuota y los costos de los servicios externos.
- Evaluar el comportamiento con un volumen mayor y con consultas reales antes de ponerlo en producción.

## 12. Conclusión

El proyecto demuestra cómo combinar automatización, recuperación de información mediante RAG y un agente de IA para asistir en la atención de consultas frecuentes de un emprendimiento gastronómico.

La solución permite responder preguntas utilizando una base de conocimiento propia, registrar solicitudes que requieren intervención humana y centralizar errores para facilitar su seguimiento. Las pruebas realizadas validan el funcionamiento del prototipo dentro del alcance definido, mientras que las limitaciones identificadas permiten establecer los próximos pasos necesarios para una implementación productiva.



