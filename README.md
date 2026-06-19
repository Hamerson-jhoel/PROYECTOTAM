# PROYECTOTAM
# 🤖 PROYECTO_TAM — SmartRecruit AI
### Sistema Inteligente de Matching y Selección de Talento

---

## 📌 Descripción General

SmartRecruit AI es un sistema de selección de talento que combina inteligencia artificial, automatización de flujos y una interfaz web interactiva. El sistema permite a un reclutador describir una vacante en lenguaje natural y obtener automáticamente un ranking de los mejores candidatos de una base de datos de 100 profesionales almacenada en Google Sheets. Además, cuenta con un chat inteligente para consultar estadísticas de la base de datos y con la opción de exportar el historial de conversación por correo electrónico.

El proyecto integra tres tecnologías principales:
- **n8n**: plataforma de automatización de flujos de trabajo (workflows).
- **Streamlit**: framework de Python para construir interfaces web interactivas.
- **Google Gemini**: modelo de lenguaje de inteligencia artificial de Google.

---

## 🧠 ¿Qué es n8n?

n8n es una herramienta de automatización de flujos de trabajo de código abierto. Permite conectar diferentes servicios y APIs mediante nodos visuales, sin necesidad de escribir código en la mayoría de los casos. En este proyecto, n8n actúa como el motor central de procesamiento: recibe las peticiones desde Streamlit, las analiza con inteligencia artificial, consulta Google Sheets y devuelve los resultados.

Cada "nodo" en n8n representa una acción (recibir datos, procesar, enviar correo, llamar a una IA, etc.). Los nodos se conectan entre sí formando un "workflow" o flujo de trabajo.

---

## 🖥️ ¿Qué es Streamlit?

Streamlit es una librería de Python que permite crear aplicaciones web interactivas de forma rápida, escribiendo solo código Python. En este proyecto, Streamlit es la interfaz visual que ve el usuario: el formulario de vacantes, el chat y la opción de exportar el historial por correo.

---

## 🔗 ¿Cómo se conecta Streamlit con n8n?

La conexión entre Streamlit y n8n se realiza mediante un **Webhook** y el protocolo **HTTP POST**. Cuando el usuario realiza una acción en Streamlit (analizar una vacante, hacer una pregunta al chat, o exportar el historial), Streamlit envía una petición HTTP POST al webhook de n8n con los datos en formato JSON.

### ¿Qué es JSON?

JSON (JavaScript Object Notation) es un formato de texto ligero para intercambiar datos entre sistemas. Es legible por humanos y fácil de procesar por máquinas. En este proyecto, todos los datos que viajan entre Streamlit y n8n están en formato JSON. Por ejemplo:

```json
{
  "action": "vacante",
  "vacante": "Buscamos un Ingeniero Electrónico con experiencia en Python..."
}
```

El campo `action` indica qué rama del flujo debe ejecutarse en n8n. Los valores posibles son `vacante`, `chat` y `export_chat`.

---

## 🗂️ Estructura del Proyecto

El proyecto tiene dos workflows en n8n:

1. **Workflow Principal** (`proyecto`): maneja las tres ramas del sistema.
2. **Sub-Workflow** (`My Sub-Workflow 1`): extrae y calcula estadísticas de la base de datos para el agente de chat.

---

## 🔄 Workflow Principal — Flujo Completo

### Diagrama de Conexiones

```
Webhook1
    └──> Switch
              ├── [acción = "vacante"]     ──> AI Agent1 ──> Get row(s) in sheet ──> Code in JavaScript1 ──> Respond to Webhook
              ├── [acción = "chat"]        ──> AI Agent  ──> Code in JavaScript2  ──> Respond to Webhook1
              └── [acción = "export_chat"] ──> Code in JavaScript ──> Send a message2 ──> Respond to Webhook2

AI Agent (chat) tiene conectados como subcomponentes:
    ├── Google Gemini Chat Model1  (modelo de lenguaje)
    ├── Simple Memory              (memoria de conversación)
    └── Call n8n Workflow Tool     (llama al Sub-Workflow de estadísticas)
```

---

## 📦 Descripción de Cada Nodo — Workflow Principal

---

### 1. 🔗 Webhook1

**Tipo:** `n8n-nodes-base.webhook`

**¿Para qué sirve?**
Es el punto de entrada del sistema. Escucha peticiones HTTP POST desde Streamlit y las pasa al resto del flujo. Es como la "puerta de entrada" del sistema.

**Configuración:**
- **Método HTTP:** POST
- **Path:** `vacante-ai`
- **Modo de respuesta:** `responseNode` (la respuesta la mandan los nodos "Respond to Webhook" al final de cada rama)

**URL de producción generada:**
```
https://[usuario].app.n8n.cloud/webhook/vacante-ai
```

Esta URL es la que se pega en el sidebar de Streamlit como "Webhook n8n".

---

### 2. 🔀 Switch

**Tipo:** `n8n-nodes-base.switch`

**¿Para qué sirve?**
Recibe los datos del Webhook y los enruta hacia la rama correcta según el campo `action` que envía Streamlit. Funciona como un semáforo: lee el campo `action` del JSON y decide qué camino seguir.

**Configuración:**
- **Modo:** Rules
- **Campo evaluado:** `{{$json.body.action}}` (en modo Expression)

| Regla | Condición | Rama destino |
|---|---|---|
| Routing Rule 1 | `action` es igual a `vacante` | AI Agent1 (análisis de vacante) |
| Routing Rule 2 | `action` es igual a `chat` | AI Agent (chat ATS) |
| Routing Rule 3 | `action` es igual a `export_chat` | Code in JavaScript (exportar correo) |

---

## 🟢 RAMA 1 — Análisis de Vacante

---

### 3. 🤖 AI Agent1 (Extractor de Vacante)

**Tipo:** `@n8n/n8n-nodes-langchain.agent`

**¿Para qué sirve?**
Recibe la descripción de la vacante en lenguaje natural escrita por el reclutador y la convierte en un objeto JSON estructurado con los campos: cargo, skills, experiencia, inglés, modalidad y salario. Usa el modelo Google Gemini para entender el texto.

**Modelo conectado:** Google Gemini Chat Model4 (`gemini-2.0-flash-lite`)

**Prompt configurado (User Message):**
```
Eres un sistema de extracción de datos.

Devuelve SOLO JSON válido.

⚠️ REGLAS OBLIGATORIAS:
NO uses markdown
NO uses texto adicional
NO uses claves con espacios
NO uses números como índices en arrays
NO uses formato Python
SOLO JSON válido estándar

📌 FORMATO EXACTO:
{
  "cargo": "string",
  "skills": ["string"],
  "experiencia": number,
  "ingles": "A1|A2|B1|B2|C1|C2",
  "modalidad": "Presencial|Remoto|Híbrido",
  "salario": number
}

📌 IMPORTANTE:
skills debe ser array real JSON ["python", "tensorflow"]
todo en minúscula en skills
ingles en mayúscula
modalidad capitalizada correctamente
experiencia número entero

📌 VACANTE:
{{ $json.body.vacante }}

SI NO PUEDES CUMPLIRLO, RESPONDE:
{}
```

**¿Por qué este prompt?**
Se instruye al agente a devolver SOLO JSON sin texto adicional ni formato Markdown, porque el siguiente nodo (Code in JavaScript1) necesita parsear ese JSON. Si el agente devuelve texto libre, el parseo falla.

---

### 4. 📊 Get row(s) in sheet (Rama 1)

**Tipo:** `n8n-nodes-base.googleSheets`

**¿Para qué sirve?**
Lee todos los candidatos de la hoja de cálculo de Google Sheets. Devuelve cada fila como un objeto JSON que el siguiente nodo puede procesar.

**Configuración:**
- **Credencial:** Google Sheets OAuth2 API
- **Documento:** SmartRecruitAI_100_Candidatos
- **Hoja:** candidatos
- **Operación:** Get Row(s) (sin filtros, trae todos los 100 candidatos)

**Columnas de la base de datos:**
`nombre`, `profesion`, `experiencia_anios`, `habilidades`, `universidad`, `ciudad`, `ingles`, `certificaciones`, `expectativa_salarial`, `modalidad`

---

### 5. 💻 Code in JavaScript1 (Ranking de Candidatos)

**Tipo:** `n8n-nodes-base.code`

**¿Para qué sirve?**
Es el corazón del sistema de matching. Recibe el perfil de la vacante (extraído por AI Agent1) y los 100 candidatos (leídos de Google Sheets), y calcula un puntaje para cada candidato basándose en 6 criterios ponderados. Devuelve el Top 10 de candidatos más compatibles.

**Algoritmo de scoring (100 puntos totales):**

| Criterio | Peso | Descripción |
|---|---|---|
| Skills/Habilidades | 35 pts | Coincidencias entre skills del candidato y los requeridos |
| Experiencia | 25 pts | Proporcional a los años requeridos vs los que tiene el candidato |
| Inglés | 10 pts | Nivel del candidato vs nivel requerido (escala A1=1 a C2=6) |
| Universidad | 5 pts | Puntaje por ranking universitario colombiano |
| Profesión | 20 pts | Coincidencia entre la profesión del candidato y el cargo buscado |
| Certificaciones | 5 pts | Si el candidato tiene certificaciones, suma 5 puntos |

Solo se incluyen candidatos con puntaje mayor o igual a 50.

**Código completo comentado:**

```javascript
// Obtiene el texto de salida del AI Agent1. Si no hay nada, usa cadena vacía.
let raw = $('AI Agent1').first().json.output || "";

// Elimina bloques de código markdown tipo ```json que Gemini a veces agrega
raw = raw
  .replace(/```json/gi, "")   // Elimina la apertura del bloque markdown
  .replace(/```/g, "")        // Elimina el cierre del bloque markdown
  .replace(/^[^\{]*/, "")     // Elimina todo lo que haya antes del primer {
  .replace(/[^\}]*$/, "")     // Elimina todo lo que haya después del último }
  .trim();                     // Elimina espacios en blanco al inicio y final

let vacante;

// Intenta convertir el texto limpio en objeto JavaScript
try {
  vacante = JSON.parse(raw);  // Parsea el JSON
} catch (e) {
  // Si el JSON está malformado, devuelve un error controlado
  return [{ json: { error: true, mensaje: "JSON inválido", raw: raw } }];
}

// Función que normaliza los nombres de habilidades para comparación
// Convierte variantes de escritura al término estándar
function normalizarSkill(skill) {
  skill = String(skill)
    .toLowerCase()                           // Todo a minúsculas
    .normalize("NFD")                        // Separa caracteres y sus acentos
    .replace(/[\u0300-\u036f]/g, "")        // Elimina los acentos
    .trim();                                 // Elimina espacios

  // Diccionario de aliases: variante -> término estándar
  const alias = {
    "piton": "python",
    "python3": "python",
    "py": "python",
    "esp-32": "esp32",
    "internet of things": "iot",
    "machine learning": "ml",
    "aprendizaje automatico": "ml",
    "inteligencia artificial": "ia",
    "artificial intelligence": "ia"
  };

  return alias[skill] || skill;
}

// Construye el objeto vacante normalizado a partir del JSON extraído por Gemini
vacante = {
  cargo: String(vacante.cargo || vacante.carga || "").trim(),  // Extrae el cargo
  skills: Array.isArray(vacante.skills)                        // Normaliza skills
    ? vacante.skills.map(s => normalizarSkill(s))
    : Array.isArray(vacante.habilidades)
      ? vacante.habilidades.map(s => normalizarSkill(s))
      : [],
  experiencia: Number(vacante.experiencia || 0),               // Años requeridos
  ingles: String(vacante.ingles || "").toUpperCase().trim(),   // Nivel de inglés
  modalidad: String(vacante.modalidad || "").trim(),           // Modalidad de trabajo
  salario: Number(vacante.salario || vacante.sueldo || 0)      // Salario esperado
};

// Obtiene todos los candidatos que vienen de Google Sheets
const candidatos = $input.all();

// Tabla de conversión de nivel de inglés a número para comparar
const niveles = { A1: 1, A2: 2, B1: 3, B2: 4, C1: 5, C2: 6 };

// Tabla de puntajes por universidad colombiana (sobre 100)
const puntajesUniversidad = {
  "Universidad Nacional de Colombia": 100,
  "Universidad de los Andes": 98,
  "Universidad de Antioquia": 95,
  "Pontificia Universidad Javeriana": 94,
  "Universidad del Valle": 93,
  "Universidad EAFIT": 92,
  "Universidad Industrial de Santander": 90,
  "Universidad del Norte": 88,
  "Universidad Distrital Francisco José de Caldas": 86,
  "Universidad Tecnológica de Pereira": 85,
  "Universidad de Nariño": 84,
  "Universidad de Caldas": 82,
  "Universidad del Cauca": 80,
  "Universidad de Cartagena": 79,
  "Universidad Pedagógica Nacional": 78,
  "Universidad Libre": 75,
  "Universidad Cooperativa de Colombia": 73,
  "Universidad Santo Tomás": 72,
  "Universidad Central": 70
};

// Calcula el puntaje de afinidad entre la profesión del candidato y el cargo requerido
function scoreProfesion(profesion, cargo) {
  profesion = String(profesion || "").toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g, "");
  cargo = String(cargo || "").toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g, "");

  // Coincidencia directa por área: score máximo 20
  if (profesion.includes("electron") && cargo.includes("electron")) return 20;
  if (profesion.includes("telecom") && cargo.includes("telecom")) return 20;
  if (profesion.includes("sistemas") && cargo.includes("sistemas")) return 20;
  if (profesion.includes("software") && cargo.includes("software")) return 20;
  if (profesion.includes("ia") && cargo.includes("ia")) return 20;
  if ((profesion.includes("datos") || profesion.includes("cientifico")) && cargo.includes("datos")) return 20;
  if (profesion.includes("industrial") && cargo.includes("industrial")) return 20;

  // Coincidencia parcial entre perfiles relacionados: 15 puntos
  if (
    (profesion.includes("software") || profesion.includes("backend") || profesion.includes("frontend")) &&
    (cargo.includes("software") || cargo.includes("backend") || cargo.includes("frontend"))
  ) return 15;

  return 0;  // Sin coincidencia
}

// Convierte el texto de habilidades de un candidato en array normalizado
function normalizarSkills(texto) {
  return String(texto || "").split(",").map(s => normalizarSkill(s)).filter(s => s.length > 0);
}

// Array que acumulará el resultado de cada candidato con su puntaje
const ranking = [];

// Itera sobre cada candidato de la base de datos
for (const item of candidatos) {
  const c = item.json;   // Datos del candidato (una fila de Google Sheets)
  let score = 0;         // Puntaje acumulado, comienza en 0

  // CRITERIO 1: SKILLS (hasta 35 puntos)
  const skillsCand = normalizarSkills(c.habilidades);  // Skills del candidato
  const skillsVac = vacante.skills;                     // Skills requeridos
  let coincidencias = 0;
  for (const s of skillsCand) {
    if (skillsVac.includes(s)) coincidencias++;  // Cuenta coincidencias exactas
  }
  if (skillsVac.length > 0) {
    score += (coincidencias / skillsVac.length) * 35;  // Puntaje proporcional
  }

  // CRITERIO 2: EXPERIENCIA (hasta 25 puntos)
  const exp = Number(c.experiencia_anios || 0);    // Años del candidato
  const expReq = Number(vacante.experiencia || 0); // Años requeridos
  if (expReq > 0) {
    score += Math.min(exp / expReq, 1) * 25;  // Proporcional, máximo 25
  } else {
    score += 25;  // Sin requisito = puntaje máximo para todos
  }

  // CRITERIO 3: INGLÉS (hasta 10 puntos)
  const nivCand = niveles[String(c.ingles || "").toUpperCase()] || 1;
  const nivReq = niveles[String(vacante.ingles || "").toUpperCase()] || 1;
  score += vacante.ingles ? Math.min(nivCand / nivReq, 1) * 10 : 10;

  // CRITERIO 4: UNIVERSIDAD (hasta 5 puntos)
  const puntUni = puntajesUniversidad[c.universidad] || 70;  // Default 70 si no está en tabla
  score += (puntUni / 100) * 5;

  // CRITERIO 5: PROFESIÓN (hasta 20 puntos)
  score += scoreProfesion(c.profesion, vacante.cargo);

  // CRITERIO 6: CERTIFICACIONES (5 puntos)
  if (c.certificaciones && String(c.certificaciones).toLowerCase() !== "ninguna") {
    score += 5;
  }

  // Agrega el candidato con su puntaje al ranking
  ranking.push({
    nombre: c.nombre,
    profesion: c.profesion,
    experiencia: exp,
    ingles: c.ingles,
    universidad: c.universidad,
    habilidades: c.habilidades,
    score_profesion: scoreProfesion(c.profesion, vacante.cargo),
    puntaje: Number(score.toFixed(2))  // Redondea a 2 decimales
  });
}

// Ordena el ranking de mayor a menor puntaje
ranking.sort((a, b) => b.puntaje - a.puntaje);

// Toma solo los primeros 10 candidatos
const top10 = ranking.slice(0, 10);

// Filtra candidatos que no alcanzaron el puntaje mínimo de 50
const PUNTAJE_MINIMO = 50;
const top10Filtrado = top10.filter(c => c.puntaje >= PUNTAJE_MINIMO);

// Si ningún candidato supera el mínimo, devuelve mensaje informativo
if (top10Filtrado.length === 0) {
  return [{
    json: {
      vacante,
      total_candidatos: 0,
      top10: [],
      mensaje: `No se encontraron candidatos que cumplan con las condiciones de esta vacante. Se revisaron ${ranking.length} perfiles y ninguno superó el puntaje mínimo requerido.`
    }
  }];
}

// Devuelve el objeto final con la vacante normalizada, el total y el top 10
return [{
  json: {
    vacante,                           // Perfil extraído de la vacante
    total_candidatos: ranking.length,  // Total de candidatos evaluados
    top10: top10Filtrado,              // Top 10 candidatos más compatibles
    mensaje: null                      // null = sin mensajes de error
  }
}];
```

---

### 6. ↩️ Respond to Webhook (Rama 1)

**Tipo:** `n8n-nodes-base.respondToWebhook`

**¿Para qué sirve?**
Devuelve el resultado del ranking a Streamlit como respuesta HTTP con el JSON completo que incluye la vacante normalizada, el total de candidatos y el Top 10.

**Configuración:**
- **Respond With:** First incoming item

---

## 🟡 RAMA 2 — Chat ATS

---

### 7. 🤖 AI Agent (Agente de Chat)

**Tipo:** `@n8n/n8n-nodes-langchain.agent`

**¿Para qué sirve?**
Es el agente conversacional del sistema. Responde preguntas sobre la base de candidatos en lenguaje natural. Antes de responder cualquier pregunta, consulta obligatoriamente el Sub-Workflow de estadísticas mediante el tool `Call n8n Workflow Tool`.

**Componentes conectados:**
- **Google Gemini Chat Model1:** modelo de IA (`gemini-2.0-flash-lite`, temperatura 0.4)
- **Simple Memory:** memoria de conversación (últimas 10 interacciones)
- **Call n8n Workflow Tool:** tool que ejecuta el Sub-Workflow de estadísticas

**System Message (en Fixed):**
```
Eres un analista de talento de SmartRecruit AI.

Tu única fuente de información es el tool "Call n8n Workflow Tool" que contiene 
las estadísticas completas de la base de 100 candidatos.

REGLAS OBLIGATORIAS:
* Llama SIEMPRE al tool antes de responder cualquier pregunta.
* Nunca inventes datos.
* Si no encuentras el dato en el tool, responde: "No tengo esa información".
* Responde siempre en español.
* Sé breve, claro y profesional.

EL TOOL CONTIENE:
- Total de candidatos
- Profesiones y cantidad por cada una
- Universidades y candidatos por universidad
- Experiencia: promedio, mayores y menores de 5 y 10 años
- Niveles de inglés (A1, A2, B1, B2, C1, C2)
- Modalidades (Presencial, Híbrido, Remoto)
- Top 10 habilidades más comunes
- Certificaciones
- Salarios: promedio, mínimo y máximo

CAPACIDADES:
1. Contar candidatos por profesión o tipo.
2. Buscar candidatos por habilidades.
3. Estadísticas generales de la base.
4. Filtrar por inglés, universidad, experiencia o modalidad.
5. Salarios promedio y rangos.

EJEMPLOS:
Usuario: ¿Cuántos ingenieros electrónicos hay?
Respuesta: Hay X ingenieros electrónicos registrados.

Usuario: ¿Cuántos candidatos saben Python?
Respuesta: Hay X candidatos con la habilidad Python.

Usuario: ¿Hay médicos?
Respuesta: No hay candidatos para esa categoría.
```

**Prompt / User Message (en Expression):**
```
={{ $json.body.pregunta }}
```

---

### 8. 🧠 Simple Memory (Memoria del Agente)

**Tipo:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`

**¿Para qué sirve?**
Permite al agente de chat recordar las preguntas y respuestas anteriores dentro de la misma sesión. Sin memoria, cada pregunta sería independiente y el agente no podría mantener una conversación coherente.

**Configuración:**
- **Session ID:** Define below → `smartrecruit_session` (en Fixed)
- **Context Window Length:** 10 (recuerda las últimas 10 interacciones)

---

### 9. 🔧 Call n8n Workflow Tool

**Tipo:** `@n8n/n8n-nodes-langchain.toolWorkflow`

**¿Para qué sirve?**
Es la herramienta que el AI Agent usa para consultar datos. Cuando el agente necesita responder una pregunta, llama a este tool que ejecuta el Sub-Workflow y devuelve las estadísticas completas de los 100 candidatos.

**Configuración:**
- **Description (en Fixed):**
```
Llama a este tool SIEMPRE que el usuario pregunte cualquier cosa sobre candidatos, 
profesiones, universidades, habilidades, experiencia, inglés, salarios o modalidades. 
Contiene estadísticas completas de los 100 candidatos.
```
- **Source:** Database
- **Workflow:** My Sub-Workflow 1

---

### 10. 💻 Code in JavaScript2 (Limpieza de Respuesta)

**Tipo:** `n8n-nodes-base.code`

**¿Para qué sirve?**
Limpia la respuesta del agente eliminando comillas dobles que podrían romper la estructura JSON de la respuesta enviada a Streamlit.

**Código comentado:**
```javascript
// Toma el primer item de entrada y extrae el campo "output" del agente
// Reemplaza todas las comillas dobles (") por comillas simples (')
// para evitar que rompan la estructura JSON de la respuesta
return {
  output: items[0].json.output.replace(/"/g, "'")
};
```

---

### 11. ↩️ Respond to Webhook1 (Rama 2)

**Tipo:** `n8n-nodes-base.respondToWebhook`

**¿Para qué sirve?**
Devuelve la respuesta del agente a Streamlit en formato JSON con el campo `answer` que el chat de Streamlit muestra al usuario.

**Configuración:**
- **Respond With:** JSON
- **Body (en Expression):**
```javascript
={{ { "answer": $json.output } }}
```

---

## 🔵 RAMA 3 — Exportar Chat por Correo

---

### 12. 💻 Code in JavaScript (Formateador de Correo)

**Tipo:** `n8n-nodes-base.code`

**¿Para qué sirve?**
Recibe el historial de conversación del chat desde Streamlit y lo convierte en un correo electrónico con formato HTML visual, con los mensajes del usuario en azul claro y los del asistente en gris.

**Código completo comentado:**

```javascript
// Extrae el body de la petición HTTP recibida desde Streamlit
// (el body contiene: email, action, chat_history)
const data = $input.first().json.body;

// Extrae el correo electrónico del destinatario
const email = data.email;

// Extrae el historial de chat. Si no existe, usa array vacío
const historial = data.chat_history || [];

// Si no hay mensajes en el historial, devuelve mensaje informativo
// pero siempre incluye el email para que Gmail pueda enviarlo
if (historial.length === 0) {
  return [{ json: { 
    email: email, 
    answer: "<p>No hay conversación para exportar.</p>" 
  }}];
}

// Variable que acumulará el HTML de todos los mensajes
let resumen = "";

// Itera sobre cada mensaje del historial
for (const msg of historial) {
  // Define la etiqueta según quién escribió el mensaje
  const rol = msg.role === "user" ? "👤 Usuario" : "🤖 Asistente";
  
  // Define el color de fondo según el rol:
  // Azul claro (#e8f0fe) para mensajes del usuario
  // Gris claro (#f1f3f4) para mensajes del asistente
  const bg = msg.role === "user" ? "#e8f0fe" : "#f1f3f4";
  
  // Construye un bloque HTML para cada mensaje con estilo inline
  resumen += `<div style="background:${bg};padding:10px;margin-bottom:8px;border-radius:6px">
    <strong>${rol}:</strong><br>${msg.content}
  </div>`;
}

// Envuelve todos los bloques en un HTML completo con título
const html = `<h2>Resumen del Chat - SmartRecruit AI</h2>${resumen}`;

// Devuelve el objeto con el email y el HTML del resumen
// Este objeto lo usará el nodo de Gmail en el siguiente paso
return [{ json: { email: email, answer: html } }];
```

---

### 13. 📧 Send a message2 (Gmail)

**Tipo:** `n8n-nodes-base.gmail`

**¿Para qué sirve?**
Envía el correo electrónico con el resumen del chat al destinatario indicado, usando la cuenta de Gmail autenticada con OAuth2.

**Configuración:**
- **Credencial:** Gmail OAuth2 API (autenticada con Google Sign In)
- **Resource:** Message
- **Operation:** Send
- **To (Expression):** `{{$json.email}}`
- **Subject (Fixed):** `Resumen del chat - SmartRecruit AI`
- **Email Type:** HTML
- **Message (Expression):** `{{$json.answer}}`

---

### 14. ↩️ Respond to Webhook2 (Rama 3)

**Tipo:** `n8n-nodes-base.respondToWebhook`

**¿Para qué sirve?**
Devuelve una respuesta de confirmación a Streamlit indicando que el correo fue enviado exitosamente. Streamlit muestra el mensaje verde de confirmación al recibir este `status: ok`.

**Configuración:**
- **Respond With:** JSON (Fixed)
- **Body:**
```json
{ "status": "ok" }
```

---

## 🔄 Sub-Workflow — My Sub-Workflow 1

Este es un workflow secundario que el AI Agent llama como tool. Su función es leer todos los candidatos de Google Sheets y calcular estadísticas completas de la base de datos.

### Diagrama de Conexiones

```
When Executed by Another Workflow ──> Get row(s) in sheet ──> Code in JavaScript
```

---

### S1. ▶️ When Executed by Another Workflow

**Tipo:** `n8n-nodes-base.executeWorkflowTrigger`

**¿Para qué sirve?**
Es el trigger del sub-workflow. Se activa cuando el nodo `Call n8n Workflow Tool` del workflow principal lo invoca. No tiene webhook propio; solo puede ser llamado por otro workflow de n8n.

**Configuración:**
- **Input data mode:** Accept All Data

---

### S2. 📊 Get row(s) in sheet (Sub-Workflow)

**Tipo:** `n8n-nodes-base.googleSheets`

**¿Para qué sirve?**
Lee todos los 100 candidatos de Google Sheets y los pasa al nodo de código para calcular las estadísticas.

**Configuración:**
- **Credencial:** Google Sheets OAuth2 API
- **Documento:** base datos
- **Hoja:** candidatos
- **Operación:** Get Row(s) sin filtros

---

### S3. 💻 Code in JavaScript (Estadísticas Completas)

**Tipo:** `n8n-nodes-base.code`

**¿Para qué sirve?**
Procesa los 100 candidatos y calcula estadísticas completas agrupadas en 8 categorías. Este resumen es lo que el AI Agent recibe cuando llama al tool para responder preguntas del usuario.

**Código completo comentado:**

```javascript
// Obtiene todos los candidatos de Google Sheets como array de objetos JavaScript
const candidatos = $input.all().map(i => i.json);
const total = candidatos.length;  // Total de candidatos en la base

// ---- 1. PROFESIONES ----

// Objeto para contar candidatos por cada profesión exacta
const profesiones = {};
for (const c of candidatos) {
  const prof = (c.profesion || "SIN_PROFESION").trim();
  profesiones[prof] = (profesiones[prof] || 0) + 1;  // Incrementa el contador
}

// Objeto para agrupar profesiones por área general normalizada
const tiposProfesion = {};
for (const c of candidatos) {
  let tipo = (c.profesion || "").toLowerCase()
    .normalize("NFD").replace(/[\u0300-\u036f]/g, "").trim();  // Elimina acentos

  // Clasifica la profesión en un área general según palabras clave
  if (tipo.includes("software")) tipo = "Ingeniería de Software";
  else if (tipo.includes("datos") || tipo.includes("data")) tipo = "Datos / Ciencia de Datos";
  else if (tipo.includes("ia") || tipo.includes("inteligencia")) tipo = "Inteligencia Artificial";
  else if (tipo.includes("electroni")) tipo = "Ingeniería Electrónica";
  else if (tipo.includes("sistemas")) tipo = "Ingeniería de Sistemas";
  else if (tipo.includes("telecom")) tipo = "Telecomunicaciones";
  else if (tipo.includes("industrial")) tipo = "Ingeniería Industrial";
  else if (tipo.includes("backend")) tipo = "Desarrollo Backend";
  else if (tipo.includes("frontend")) tipo = "Desarrollo Frontend";
  else tipo = "Otra";

  tiposProfesion[tipo] = (tiposProfesion[tipo] || 0) + 1;
}

// ---- 2. UNIVERSIDADES ----

const universidades = {};
for (const c of candidatos) {
  const uni = (c.universidad || "SIN_UNIVERSIDAD").trim();
  universidades[uni] = (universidades[uni] || 0) + 1;  // Cuenta por universidad
}
const totalUniversidades = Object.keys(universidades).length;  // Cuántas universidades únicas

// ---- 3. EXPERIENCIA ----

let expMas10 = 0;    // Candidatos con 10 o más años de experiencia
let expMenos10 = 0;  // Candidatos con menos de 10 años
let expMas5 = 0;     // Candidatos con 5 o más años
let expMenos5 = 0;   // Candidatos con menos de 5 años
let totalExp = 0;    // Suma total para calcular promedio

for (const c of candidatos) {
  const exp = Number(c.experiencia_anios || 0);
  totalExp += exp;
  if (exp >= 10) expMas10++; else expMenos10++;
  if (exp >= 5) expMas5++; else expMenos5++;
}

// Promedio de experiencia con 1 decimal
const promedioExperiencia = total > 0 ? Number((totalExp / total).toFixed(1)) : 0;

// ---- 4. INGLÉS ----

const niveles = {};
for (const c of candidatos) {
  const nivel = (c.ingles || "SIN_NIVEL").trim().toUpperCase();  // Normaliza a mayúsculas
  niveles[nivel] = (niveles[nivel] || 0) + 1;
}

// Agrupa en tres categorías para facilitar análisis
const nivelesAltos = (niveles["C1"] || 0) + (niveles["C2"] || 0);   // Nivel avanzado
const nivelesMedios = (niveles["B1"] || 0) + (niveles["B2"] || 0);  // Nivel intermedio
const nivelesBajos = (niveles["A1"] || 0) + (niveles["A2"] || 0);   // Nivel básico

// ---- 5. MODALIDAD ----

const modalidades = {};
for (const c of candidatos) {
  const mod = (c.modalidad || "SIN_MODALIDAD").trim();
  modalidades[mod] = (modalidades[mod] || 0) + 1;  // Presencial, Híbrido, Remoto
}

// ---- 6. HABILIDADES ----

const habilidades = {};
for (const c of candidatos) {
  const skills = (c.habilidades || "").split(",");  // Las skills vienen separadas por comas
  for (const s of skills) {
    const skill = s.trim();
    if (skill) habilidades[skill] = (habilidades[skill] || 0) + 1;
  }
}

// Construye el Top 10 de habilidades más comunes ordenando de mayor a menor
const top10Habilidades = Object.entries(habilidades)
  .sort((a, b) => b[1] - a[1])  // Ordena descendente por frecuencia
  .slice(0, 10)                  // Toma los 10 primeros
  .reduce((acc, [k, v]) => { acc[k] = v; return acc; }, {});  // Convierte a objeto

// ---- 7. CERTIFICACIONES ----

let conCertificacion = 0;
let sinCertificacion = 0;
const tipoCertificaciones = {};

for (const c of candidatos) {
  const cert = (c.certificaciones || "").trim().toLowerCase();
  if (!cert || cert === "ninguna" || cert === "sin certificacion") {
    sinCertificacion++;  // Sin certificación
  } else {
    conCertificacion++;  // Con certificación
    tipoCertificaciones[c.certificaciones.trim()] =
      (tipoCertificaciones[c.certificaciones.trim()] || 0) + 1;
  }
}

// ---- 8. SALARIOS ----

// Extrae salarios numéricos y filtra los que sean mayores a 0
const salarios = candidatos
  .map(c => Number(c.expectativa_salarial || 0))
  .filter(s => s > 0);

const promedioSalario = salarios.length > 0
  ? Math.round(salarios.reduce((a, b) => a + b, 0) / salarios.length)  // Promedio redondeado
  : 0;
const salarioMin = salarios.length > 0 ? Math.min(...salarios) : 0;  // Mínimo
const salarioMax = salarios.length > 0 ? Math.max(...salarios) : 0;  // Máximo

// ---- RESPUESTA FINAL ----
// Devuelve un objeto JSON estructurado con todas las estadísticas
// Este es el objeto que recibe el AI Agent como respuesta del tool

return [{
  json: {
    resumen_general: {
      total_candidatos: total,
      total_universidades_diferentes: totalUniversidades,
      promedio_experiencia_anios: promedioExperiencia
    },
    profesiones: {
      detalle_por_profesion: profesiones,          // {"Ingeniero Electrónico": 5, ...}
      agrupado_por_tipo: tiposProfesion,           // {"Ingeniería Electrónica": 8, ...}
      total_profesiones_diferentes: Object.keys(profesiones).length
    },
    universidades: {
      total_universidades: totalUniversidades,
      candidatos_por_universidad: universidades    // {"Universidad Nacional": 12, ...}
    },
    experiencia: {
      con_10_o_mas_anios: expMas10,
      con_menos_de_10_anios: expMenos10,
      con_5_o_mas_anios: expMas5,
      con_menos_de_5_anios: expMenos5,
      promedio_anios: promedioExperiencia
    },
    ingles: {
      distribucion_por_nivel: niveles,    // {"A1": 17, "B2": 20, "C2": 19, ...}
      nivel_alto_C1_C2: nivelesAltos,
      nivel_medio_B1_B2: nivelesMedios,
      nivel_basico_A1_A2: nivelesBajos
    },
    modalidades: {
      distribucion: modalidades           // {"Presencial": 34, "Híbrido": 39, "Remoto": 27}
    },
    habilidades: {
      top_10_habilidades: top10Habilidades,       // {"Spring Boot": 39, "Python": 36, ...}
      total_habilidades_diferentes: Object.keys(habilidades).length
    },
    certificaciones: {
      con_certificacion: conCertificacion,
      sin_certificacion: sinCertificacion,
      detalle_certificaciones: tipoCertificaciones  // {"Oracle SQL Associate": 14, ...}
    },
    salarios: {
      promedio: promedioSalario,   // 5760000
      minimo: salarioMin,          // 2500000
      maximo: salarioMax           // 10000000
    }
  }
}];
```

---

## 🖥️ Frontend — Streamlit (app.py)

Streamlit es la interfaz visual del sistema. Se ejecuta como una aplicación web en Python y se comunica con n8n mediante peticiones HTTP POST en formato JSON.

### Dependencias instaladas

```bash
pip install streamlit plotly requests pyngrok openpyxl pandas
```

### Estructura general de la app

```
app.py
├── Configuración de página (set_page_config)
├── Estilos CSS personalizados
├── Header: título y subtítulo
├── Sidebar
│   ├── Campo Webhook URL (input de texto)
│   └── Selector de módulo (radio: Vacante / Chat ATS)
├── Módulo 1: 👤 Vacante
│   ├── Text area para escribir la vacante
│   ├── Botón "Analizar Candidatos"
│   └── Función mostrar_resultado_vacante()
│       ├── Métricas (total, mejor match, puntaje, cargo)
│       ├── Perfil detectado + Skills
│       ├── Top 3 con medallas
│       ├── Gráfico de barras (Plotly)
│       └── Tabla completa (DataFrame)
└── Módulo 2: 💬 Chat ATS
    ├── Checkbox de exportar por correo
    │   ├── Input de email
    │   └── Botón "Enviar resumen" → action: export_chat
    ├── Historial de chat (session_state)
    ├── Chat input → action: chat
    └── Botón "Limpiar conversación"
```

### Cómo Streamlit envía datos a n8n

**Módulo Vacante:**
```python
respuesta = requests.post(
    WEBHOOK_URL,
    json={"action": "vacante", "vacante": vacante},
    timeout=120    # 2 minutos de espera máxima
)
```

**Módulo Chat:**
```python
respuesta = requests.post(
    WEBHOOK_URL,
    json={"action": "chat", "pregunta": pregunta},
    timeout=200    # 3.3 minutos de espera (el agente puede tardar)
)
```

**Exportar Chat:**
```python
resp = requests.post(
    WEBHOOK_URL,
    json={
        "action": "export_chat",
        "email": email,
        "chat_history": st.session_state.chat_history  # Lista de mensajes del historial
    },
    timeout=60
)
```

### Exposición pública con ngrok

Para acceder a la app desde cualquier dispositivo sin desplegar en un servidor, se usa ngrok, que crea un túnel desde internet hacia el puerto local 8501 donde corre Streamlit:

```python
from pyngrok import ngrok
url = ngrok.connect(8501)
print(url.public_url)  # URL pública tipo https://xxxx.ngrok-free.dev
```

---

## 📊 Resultados Obtenidos

### Base de datos procesada
- **100 candidatos** con 9 columnas cada uno
- **19 universidades** diferentes representadas
- **19 profesiones** distintas en la base

### Estadísticas de la base

| Métrica | Valor |
|---|---|
| Total candidatos | 100 |
| Universidades diferentes | 19 |
| Promedio de experiencia | 5.4 años |
| Candidatos nivel C2 (inglés) | 19 |
| Candidatos nivel B2 (inglés) | 20 |
| Modalidad más común | Híbrido (39) |
| Skill más frecuente | Spring Boot (39 candidatos) |
| Salario promedio | $5.760.000 COP |
| Salario mínimo | $2.500.000 COP |
| Salario máximo | $10.000.000 COP |

### Funcionalidades verificadas

- ✅ Rama 1: Análisis de vacante con ranking de candidatos funciona correctamente
- ✅ Rama 2: Chat ATS responde preguntas sobre la base de datos en lenguaje natural
- ✅ Rama 3: Exportación del historial de chat por correo electrónico en formato HTML
- ✅ Memoria conversacional: el agente recuerda las últimas 10 interacciones
- ✅ Sub-Workflow de estadísticas: extrae y pre-calcula todos los indicadores de la base

---

## 🛠️ Tecnologías Utilizadas

| Herramienta | Uso |
|---|---|
| n8n Cloud 2.25.7 | Motor de automatización y flujos |
| Google Gemini 2.0 flash-lite | Modelo de lenguaje (IA) |
| Google Sheets | Base de datos de candidatos |
| Gmail OAuth2 | Envío de correos |
| Streamlit | Interfaz web |
| Python 3.x | Lenguaje del frontend |
| Plotly Express | Gráficos interactivos |
| Pandas | Manipulación de datos |
| ngrok | Exposición pública de la app |
| Google Colab | Entorno de ejecución del frontend |

---

## 🚀 Cómo Ejecutar el Proyecto

1. Abrir Google Colab y ejecutar el notebook con el código de Streamlit
2. Copiar la URL pública generada por ngrok
3. En n8n, asegurarse de que el workflow principal esté **Published**
4. Pegar la URL del webhook de n8n en el sidebar de Streamlit como "Webhook n8n"
5. Seleccionar el módulo deseado: Vacante o Chat ATS
6. El sistema está operativo

---

*Proyecto desarrollado como sistema de demostración de integración entre IA generativa, automatización de flujos y visualización de datos aplicado a procesos de selección de talento humano.*
