# ⚽ Fulbot AI Manager

Este repositorio contiene un flujo de trabajo avanzado de **n8n** que implementa un agente de Inteligencia Artificial para automatizar la gestión de un complejo de canchas de fútbol vía WhatsApp.

El sistema utiliza un enfoque híbrido: **Google Gemini** como cerebro principal para la conversación y gestión de herramientas, y **Llama 3.3 (70B)** dedicado a la capa de seguridad (Guardrails).

## ✨ Características Principales

  * **🤖 IA Híbrida:** Utiliza **Google Gemini** para interpretar intenciones complejas y manejar la lógica del negocio, asegurando respuestas rápidas y precisas.
  * **🛡️ Seguridad Avanzada (Guardrails):** Implementa **Llama 3.3 70B** específicamente para filtrar inputs maliciosos y moderar el contenido antes de que llegue al agente principal.
  * **📅 Gestión de Reservas Inteligente:** Verifica disponibilidad con precisión utilizando tipos de datos de rango (`tsrange`) en PostgreSQL para evitar solapamientos exactos o parciales.
  * **💸 Pagos Automáticos:** Integración con Webhooks de MercadoPago para confirmar pagos recibidos y actualizar el estado de la reserva en tiempo real.
  * **🚫 Sistema de Strikes:** Detecta intentos de manipulación (*jailbreak*) y bloquea usuarios automáticamente tras 3 advertencias ("strikes").
  * **🧠 Memoria Contextual:** Utiliza una ventana de memoria buffer y **Redis** para mantener la coherencia en la charla.
  * **⚡ Buffer de Mensajes:** Implementa **Redis** para encolar mensajes entrantes, evitando condiciones de carrera si el usuario envía múltiples mensajes rápidos.
  * **👨‍💻 Derivación a Humano:** Comando `/humano` o detección automática para pausar la IA y solicitar asistencia.

## 🛠️ Stack Tecnológico

  * **Orquestador:** [n8n](https://n8n.io/)
  * **LLM Principal:** `Google Gemini` (Chat Model).
  * **LLM Guardrails:** `llama-3.3-70b-versatile` (Vía nodo OpenAI/Groq).
  * **Base de Datos:** PostgreSQL (Con extensión `btree_gist`).
  * **Gestión de Estados:** n8n Data Tables.
  * **Cache/Buffer:** Redis.
  * **Mensajería:** WhatsApp (vía API Gateway, ej: Evolution API).
  * **Pagos:** MercadoPago API.

## 📋 Requisitos Previos

Para que este flujo funcione, necesitas:

1.  Una instancia de **n8n**.
2.  Servidor de **PostgreSQL** (con permisos para crear extensiones).
3.  Servidor de **Redis**.
4.  **API Key de Google Gemini** (PaLM/Gemini API).
5.  **API Key para Llama 3.3** (Groq o proveedor compatible con OpenAI).
6.  Cuenta de **MercadoPago** (Access Token).
7.  Una API de WhatsApp (Evolution API o similar).

## ⚙️ Configuración de Base de Datos (PostgreSQL)

⚠️ **IMPORTANTE:** El esquema ha sido actualizado para usar restricciones de exclusión (`EXCLUDE`). Asegúrate de ejecutar este script, ya que la versión anterior con `UNIQUE` no soporta rangos de tiempo complejos.

```sql
-- 1. Extensiones (Necesaria para restricciones de exclusión con enteros y rangos)
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- 2. Esquema
DROP SCHEMA IF EXISTS negocio CASCADE;
CREATE SCHEMA negocio;

-- 3. Tabla de Canchas
CREATE TABLE IF NOT EXISTS negocio.canchas (
    "NUMERO" INTEGER PRIMARY KEY,
    "FUTBOL" INTEGER,
    "PRECIO" NUMERIC(15, 2)
);

-- Datos iniciales
INSERT INTO negocio.canchas ("NUMERO", "FUTBOL", "PRECIO") 
VALUES (1, 5, 50000), (2, 7, 70000)
ON CONFLICT ("NUMERO") DO NOTHING;

-- 4. Tabla de Reservas (Con protección avanzada anti-solapamiento)
CREATE TABLE IF NOT EXISTS negocio.reservas (
    id SERIAL PRIMARY KEY,
    "TELEFONO" VARCHAR(50),
    "NUMEROCANCHA" INTEGER REFERENCES negocio.canchas("NUMERO"),
    "PRECIO" NUMERIC(15, 2),
    "SEÑA" NUMERIC(15, 2),
    "FALTANTE" NUMERIC(15, 2) GENERATED ALWAYS AS ("PRECIO" - "SEÑA") STORED,
    "FECHA" DATE,
    "HORARIO" TIME,
    "TIEMPO" NUMERIC(5, 1),
    "NOMBRE" VARCHAR(255),
    "PAGORECIBIDO" VARCHAR(10) DEFAULT 'no',
    "IDENTIFICADOR" VARCHAR(8) GENERATED ALWAYS AS (LPAD(id::TEXT, 8, '0')) STORED UNIQUE,
    "ESTADO" VARCHAR(20) DEFAULT 'ESPERA',
    "CREATED_AT" TIMESTAMP DEFAULT NOW(),
    
    -- Restricción de exclusión:
    -- Evita que se reserve la misma cancha si los rangos de tiempo se superponen.
    CONSTRAINT evitar_superposicion_canchas
    EXCLUDE USING gist (
        "NUMEROCANCHA" WITH =,
        tsrange(
            ("FECHA" + "HORARIO"), 
            ("FECHA" + "HORARIO" + ("TIEMPO" * interval '1 hour')), 
            '[)'
        ) WITH &&
    )
);
````

### 🚀 Instalación y Configuración en n8n

1.  **Importar:** Carga el archivo `Fulbot-AI-Manager.json` y los flujos de las herramientas.
2.  **Credenciales:**
      * **Google Gemini(PaLM) Api account:** Para el agente principal.
      * **OpenAi account:** Para el nodo de Guardrails (usando Llama 3.3).
      * **Postgres account:** Conexión a tu BD.
      * **Redis account:** Conexión a Redis.
      * **Bearer Auth account:** Para la API de WhatsApp.
      * **mp-prueba:** Para MercadoPago.
3.  **Data Tables:**
      * Asegúrate de tener la tabla `EstadoConersasiones` (Columns: `CHAT_ID`, `STATUS`, `FECHA`, `AVISOS`).
4.  **Tools (Sub-flujos):**
    Asegúrate de que los nodos "Call Workflow" o "Tool" en el agente apunten a los ID correctos de estos flujos:
      * `VerDisponibilidadFulbot`
      * `ReservarCanchaFulbot`
      * `RecuperarReservasFulbot`
      * `EliminarReservaFulbot`
      * `PosiblesHorarios`

### ⚠️ Lógica de Seguridad (Strikes)

1.  **Entrada:** El mensaje pasa por **Guardrails** (Llama 3.3).
2.  **Detección:** Si se detecta *jailbreak* o contenido inapropiado, se incrementa el contador de avisos en `EstadoConersasiones`.
3.  **Consecuencias:**
      * **Aviso 1-2:** Advertencia.
      * **Aviso 3:** Último aviso.
      * **Aviso 4:** Bloqueo permanente (la IA deja de responder).

#### Desarrollado con ❤️ usando n8n.
