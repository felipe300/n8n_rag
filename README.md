# Cerebro de Agente IA: RAG con n8n y Cohere

Este proyecto demuestra cómo transformar documentos estáticos (PDFs, TXT, Manuales) en un Agente de Inteligencia Artificial capaz de responder preguntas basadas en un contexto real. Utiliza una arquitectura RAG (Generación Aumentada por Recuperación) implementada sobre n8n.

Está especialmente diseñado para el sector educativo, permitiendo que docentes y administradores interactúen con normativas, guías y materiales pedagógicos de forma inteligente.

## ¿Qué aprenderás con este proyecto?

- **RAG vs. SQL**: Cuándo y cómo hacer que una IA consulte documentación estática (búsqueda semántica) frente a registros estructurados exactos (consultas relacionales).

- **Agentes basados en Herramientas (Tools)**: Configuración de un nodo `AI Agent` en n8n capaz de invocar de forma autónoma una base de datos vectorial o un motor SQL.

- **Extracción de Parámetros Dinámicos (`$fromAI`)**: Cómo usar la IA para identificar variables en el lenguaje natural (como un correo electrónico) e inyectarlas de forma segura en tus consultas SQL.

- **Gestión Segura de Contenedores**: Orquestación de redes internas aisladas en Docker para comunicar n8n con bases de datos relacionales sin hardcodear secretos.

## Requisitos Previos

- Docker y Docker Compose instalados.
- n8n (versión con nodos de IA habilitados).
- API Key de Cohere (puedes obtener una gratuita en [Cohere](https:www.cohere.com)).

## Estructura del proyecto

```bash
.
├── assets/                         # Esquemas SQL y recursos visuales
│   └── 01_employees_class_02.sql   # Script de inicialización de la base de datos
├── chocolatech-inmersion-g10/      # Documentación base (Contexto RAG)
│   ├── Manual de RH.txt
│   └── Política de Viajes.txt
├── output/                         # Carpeta para archivos procesados
├── .env                            # Variables de entorno y secretos (Ignorado por Git)
├── .env.example                    # Plantilla de configuración
├── docker-compose.yml              # Orquestación de n8n y MySQL en Docker
└── README.md
```

## Configuración del entorno

1. **Clonar el Repositorio**:

```bash
git clone https://github.com/felipe300/n8n_rag.git
cd n8n_rag
```

2. **Configura tus variables de entorno**

Copia el archivo y edita el archivo `.env`. La API Key de Cohere se agrega directamente en el nodo de n8n.

```bash
cp .env.example .env
```

3. **Inicia el Proyecto**

En la raíz del proyecto, ejecuta:

```bash
# levantar docker compose
docker compose up -d

# detener docker compose
docker compose down
```

4. **Visualizar y Administrar los Datos (Adminer)**

No necesitas instalar programas externos. Abre tu navegador en `http://localhost:8080` para acceder a **Adminer** e ingresa con estos datos para explorar tus tablas estructuradas:

- **Sistema**: `MySQL`
- **Servidor**: `mysql-db`
- **Usuario**: El valor de tu `${MYSQL_USER}` (ej. `n8n_user`)
- **Contraseña**: El valor de tu `${MYSQL_PASSWORD}`
- **Base de datos**: El valor de tu `${MYSQL_DATABASE}` (ej. `n8n_db)`

5. **Acceder a n8n**:

Abre tu navegador en `http://localhost:5678` e inicia sesión con las credenciales que definiste en tu archivo `.env` (`N8N_USER` y `N8N_PASSWORD`).

## Arquitectura del Agente

En entornos locales, la ejecución de los flujos pueden tomar más tiempo.

El agente de n8n procesa la información de los colaboradores dividiendo el conocimiento en dos grandes vertientes operadas por el modelo `command-r` de Cohere:

```text
                                 │
                                 ▼
                           ┌───────────┐
                           │ AI Agent  │ ◄─── (Memoria de Sesión)
                           └─────┬─────┘
                                 │
         ┌───────────────────────┴───────────────────────┐
         ▼                                               ▼
[¿Es una duda general/reglamentos?]           [¿Es una duda de sus datos/saldos?]
         │                                               │
         ▼                                               ▼
┌──────────────────┐                           ┌──────────────────┐
│  Vector Store    │                           │   Custom Tool    │
│  Tool (RAG)      │                           │  (MySQL Host:    │
│ (Qdrant/Pinecone)│                           │   `mysql-db`)    │
└────────┬─────────┘                           └────────┬─────────┘
         │                                               │
         └───────────────────────┬───────────────────────┘
                                 │ (Retorna Información)
                                 ▼
                       ┌──────────────────┐
                       │ Respuesta Final  │ (Personalizada en Lenguaje Natural)
                       └──────────────────┘
```

El flujo de trabajo se divide en dos fases críticas:

1. **Flujo de Ingestión (Load Data Flow)**

Se encarga de leer los archivos de la carpeta `./chocolatech-inmersion-g10`, dividirlos en fragmentos manejables y generar los embeddings multilingües con Cohere para guardarlos en una base de datos vectorial (Vector Store).

![Flujo de ingestion](./assets/n8n_rag_flow_01.png)

2. **Flujo de Consulta (Query Flow)**

Cuando el usuario hace una pregunta, el agente busca en el Vector Store los fragmentos más relevantes y utiliza un LLM para redactar una respuesta precisa basada únicamente en esos fragmentos.

![Flujo de consulta](./assets/n8n_rag_flow_02.png)

Además, se agregar las siguientes mejoras:

3. Integración IA con MySQL

Ahora el usuario debe dar su nombre y sólo consultar los datos acordes a su persona. Estos datos provienen de la base de datos (`./assets/01_employees_class_02.sql`) de la tabla `empleados`. Más aún, se mejora el prompt para obtener una mejor respuesta.

![Flow 03](./assets/n8n_rag_flow_03.png)

Nuevo prompt:

```text
"Eres el HR Buddy, asistente virtual de RR. HH. de ChocolaTech.
REGLAS:

    Responde siempre en español.
    Responde SOLO dudas relacionadas con RR. HH.
    IDENTIFICACIÓN DEL EMPLEADO:

    Si el usuario no dice quién es, pregúntale su nombre completo en la primera respuesta.
    Usa la herramienta MySQL para buscar en la tabla funcionarios usando SIEMPRE el NOMBRE COMPLETO informado por el usuario en la conversación.
    Si se encuentra: usa los saldos de vacaciones y banco de horas.
    Si no se encuentra: no inventes datos personales. Responde solo con base en las políticas generales de RR. HH. del Vector Store.
    Usa la base de conocimientos para dudas generales."
```

## Enlaces de Interés

- [n8n.io](https://n8n.io/) - Plataforma de automatización.
- [Cohere Dashboard](https://cohere.com/) - Gestión de API Keys y Modelos.
- [Manual del Colaborador y Políticas de Recursos Humanos - CHOCOLATECH](https://raw.githubusercontent.com/ericmonne/chocolatech-inmersion-g10/refs/heads/main/Manual%20de%20RH%20ChocolaTech%20LAD.txt)
- [Repositorio de Archivos CHOCOLATECH](https://github.com/ericmonne/chocolatech-inmersion-g10) - Documentación de ejemplo para inmersión.
