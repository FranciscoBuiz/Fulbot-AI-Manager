# ⚽ Fulbot AI Manager

Este repositorio contiene un flujo de trabajo avanzado de **n8n** que implementa un agente de Inteligencia Artificial para automatizar la gestión de un complejo de canchas de fútbol vía WhatsApp.

El sistema utiliza **Llama 3.3 (70B)** para interpretar el lenguaje natural, gestionar reservas, verificar pagos y bloquear usuarios abusivos automáticamente.

## ✨ Características Principales

  * **🤖 IA Conversacional:** Entiende intenciones complejas (reservar, cancelar, consultar precios) utilizando el modelo **Llama 3.3 70B** (vía nodo OpenAI compatible).
  * **📅 Gestión de Reservas:** Verifica disponibilidad en tiempo real, sugiere horarios alternativos y previene dobles reservas mediante restricciones en base de datos.
  * **💸 Pagos Automáticos:** Integración con Webhooks de MercadoPago para confirmar pagos recibidos y actualizar el estado de la reserva.
  * **🛡️ Seguridad y Moderación:** Sistema de "Guardrails" que detecta intentos de manipulación (*jailbreak*) y bloquea usuarios tras 3 advertencias ("strikes").
  * **🧠 Memoria Contextual:** Utiliza una ventana de memoria buffer y **Redis** para mantener la coherencia en la charla.
  * **⚡ Alto Rendimiento:** Implementa **Redis** para el *buffering* de mensajes entrantes, evitando condiciones de carrera si el usuario envía múltiples mensajes rápidos.
  * **👨‍💻 Derivación a Humano:** Comando `/humano` o detección automática para cambiar el estado de la conversación y detener la IA.

## 🛠️ Stack Tecnológico

  * **Orquestador:** [n8n](https://n8n.io/)
  * **LLM:** `llama-3.3-70b-versatile` (Configurado en nodo OpenAI, recomendado usar Groq).
  * **Base de Datos:** PostgreSQL (Esquema `negocio`).
  * **Gestión de Estados:** n8n Data Tables (Base de datos interna de n8n).
  * **Cache/Buffer:** Redis.
  * **Mensajería:** WhatsApp (vía API Gateway, ej: Evolution API).
  * **Pagos:** MercadoPago API.

## 📋 Requisitos Previos

Para que este flujo funcione, necesitas:

1.  Una instancia de **n8n** (Self-hosted recomendada para uso de Redis/Postgres local).
2.  Servidor de **PostgreSQL**.
3.  Servidor de **Redis**.
4.  **API Key para LLM** (Groq o proveedor compatible con OpenAI para usar Llama 3.3).
5.  Cuenta de **MercadoPago** (para Access Token).
6.  Una API de WhatsApp local o remota que envíe webhooks al n8n.

## ⚙️ Configuración de Base de Datos (PostgreSQL)

El flujo incluye un nodo de inicialización, pero asegúrate de tener este esquema en tu PostgreSQL para evitar errores iniciales.

> **Nota:** La gestión del estado de la conversación (Bot vs Humano) y el conteo de avisos se maneja internamente con **n8n Data Tables**, por lo que no requiere una tabla SQL adicional.

```sql
CREATE SCHEMA IF NOT EXISTS negocio;

-- 1. Tabla de Canchas
CREATE TABLE IF NOT EXISTS negocio.canchas (
    "NUMERO" INTEGER PRIMARY KEY,
    "FUTBOL" INTEGER,
    "PRECIO" NUMERIC(15, 2)
);

-- Datos iniciales de ejemplo
INSERT INTO negocio.canchas ("NUMERO", "FUTBOL", "PRECIO") 
VALUES (1, 5, 50000), (2, 7, 70000)
ON CONFLICT ("NUMERO") DO NOTHING;

-- 2. Tabla de Reservas
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
    
    -- Protección contra doble reserva (Unique Constraint)
    CONSTRAINT unique_reserva UNIQUE ("NUMEROCANCHA", "FECHA", "HORARIO")
);
```

### 🚀 Instalación y Configuración en n8n

1.  **Importar:** Carga el archivo `.json` del flujo en tu n8n.

2.  **Credenciales:** Configura las siguientes credenciales en n8n:

      * **Postgres account:** Conexión a tu BD `negocio`.
      * **Redis account:** Conexión a tu instancia Redis.
      * **OpenAi account:** Aquí debes colocar tu API Key (ej. de Groq) y en la configuración del nodo ajustar la *Base URL* si no usas OpenAI oficial.
      * **Bearer Auth account:** Para autenticar las peticiones hacia tu API de WhatsApp.
      * **mp-prueba:** Credenciales HTTP Bearer para consultar la API de MercadoPago.

3.  **Data Tables (Interno):**

      * El flujo utiliza una tabla interna de n8n llamada `EstadoConersasiones`. Si n8n no la crea automáticamente al importar, deberás crearla en el menú "Variables" -\> "Data Tables" con las columnas: `CHAT_ID` (Primary, String), `STATUS` (String), `FECHA` (DateTime), `AVISOS` (Number).

4.  **Sub-flujos (Tools):**
    Este flujo principal actúa como orquestador y llama a otros flujos "Tool" que deben existir en tu instancia. Asegúrate de tener los siguientes workflows creados/importados y que sus IDs coincidan o sean re-vinculados en los nodos de "AI Agent":

      * `ProximoHorarioDisponible`
      * `VerDisponibilidadFulbot`
      * `ReservarCanchaFulbot`
      * `RecuperarReservasFulbot`
      * `EliminarReserva`
      * `PosiblesHorarios`

### ⚠️ Lógica de Seguridad (Strikes)

El sistema maneja una lógica de bloqueo automático basada en el comportamiento del usuario (detectado por Guardrails o mal uso):

  * **Aviso 1 y 2:** El usuario recibe advertencias.
  * **Aviso 3:** Se envía mensaje de "Último Strike".
  * **Aviso 4+:** Se envía mensaje de "Bloqueo permanente" y el sistema deja de procesar la IA para ese número.

#### Desarrollado con ❤️ usando n8n.
