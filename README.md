# ⚽ Fulbot AI Manager

Este repositorio contiene un flujo de trabajo avanzado de **n8n** que implementa un agente de Inteligencia Artificial para automatizar la gestión de un complejo de canchas de fútbol vía WhatsApp.

El sistema utiliza **Groq** para interpretar el lenguaje natural, gestionar reservas, verificar pagos y bloquear usuarios abusivos automáticamente.

## ✨ Características Principales

* **🤖 IA Conversacional:** Entiende intenciones complejas (reservar, cancelar, consultar precios) usando Google Gemini (LangChain).
* **📅 Gestión de Reservas:** Verifica disponibilidad en tiempo real y sugiere horarios alternativos.
* **💸 Pagos Automáticos:** Integración con Webhooks de MercadoPago para confirmar señas y actualizar la base de datos automáticamente.
* **🛡️ Seguridad y Moderación:** Sistema de "Guardrails" que detecta intentos de *jailbreak*. Bloquea usuarios tras 3 "strikes" (advertencias).
* **🧠 Memoria Contextual:** Recuerda la conversación reciente para dar respuestas coherentes.
* **⚡ Alto Rendimiento:** Usa **Redis** para *buffering* de mensajes (evita respuestas duplicadas si el usuario escribe muy rápido).
* **👨‍💻 Derivación a Humano:** Comando `/humano` o detección automática de situaciones complejas para alertar al staff.

## 🛠️ Stack Tecnológico

* **Orquestador:** [n8n](https://n8n.io/)
* **LLM:** Llama-3.3-70b-versatile
* **Base de Datos:** PostgreSQL (Esquema `negocio`)
* **Cache/Buffer:** Redis
* **Mensajería:** WhatsApp (vía API Gateway, ej: Evolution API)
* **Pagos:** MercadoPago API

## 📋 Requisitos Previos

Para que este flujo funcione, necesitas:

1.  Una instancia de **n8n** (Self-hosted recomendada para uso de Redis/Postgres local).
2.  Servidor de **PostgreSQL**.
3.  Servidor de **Redis**.
4.  Cuenta de **Google Cloud** (para Gemini API Key).
5.  Cuenta de **MercadoPago** (para Access Token).
6.  Una API de WhatsApp (tipo Evolution API o similar) que envíe webhooks al n8n.

## ⚙️ Configuración de Base de Datos

El flujo incluye un nodo de inicialización, pero asegúrate de tener este esquema en tu PostgreSQL:

```sql
CREATE SCHEMA IF NOT EXISTS negocio;

-- Tabla de Canchas
CREATE TABLE IF NOT EXISTS negocio.canchas (
    "NUMERO" INTEGER PRIMARY KEY,
    "FUTBOL" INTEGER,
    "PRECIO" NUMERIC(15, 2)
);

-- Tabla de Reservas
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
    "PAGORECIBIDO" VARCHAR(10),
    "IDENTIFICADOR" VARCHAR(8) GENERATED ALWAYS AS (LPAD(id::TEXT, 8, '0')) STORED UNIQUE
);

-- Tabla de Estados de Conversación (Bot/Humano/Bloqueado)
CREATE TABLE IF NOT EXISTS negocio.estados_conversaciones (
    "CHAT_ID" VARCHAR(100) PRIMARY KEY,
    "STATUS" VARCHAR(50),
    "FECHA" TIMESTAMP WITH TIME ZONE,
    "AVISOS" INTEGER DEFAULT 0
);
```
### 🚀 Instalación
Importa el archivo .json del flujo en tu n8n.

Configura las credenciales en n8n para:

1.  Postgres account
2.  Redis account
3.  Google Gemini(PaLM) Api account
4.  Bearer Auth account (Para tu API de WhatsApp)
5.  mp-prueba (MercadoPago)

Importante: Este flujo principal llama a otros sub-flujos (Tools) que deben existir en tu n8n:

1.  ProximoHorarioDisponible
2.  VerDisponibilidadFulbot
3.  ReservarCanchaFulbot
4.  RecuperarReservasFulbot
5.  EliminarReservaFulbot
6.  PosiblesHorarios

### ⚠️ Variables de Entorno y Seguridad
El sistema maneja lógica de bloqueo. Si un usuario intenta abusar del bot:

* Aviso 1 y 2: Advertencia.
* Aviso 3: Último Strike.
* Aviso 4: Bloqueo permanente (registrado en BD).

#### Desarrollado con ❤️ usando n8n.
