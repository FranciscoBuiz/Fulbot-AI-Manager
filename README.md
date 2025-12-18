# ⚽ Fulbot AI Manager

Este repositorio contiene un ecosistema avanzado de automatización basado en **n8n** para la gestión integral de un complejo de canchas de fútbol. El sistema actúa como un agente de IA autónomo capaz de conversar con clientes vía WhatsApp, procesar reservas, gestionar pagos y administrar cancelaciones.

## 🤖 Arquitectura del Agente

El corazón del sistema es un agente de **Google Gemini** configurado con herramientas específicas (sub-flujos) para interactuar con la base de datos y servicios externos.

### 🧠 Capas de Inteligencia

* **Agente Principal:** `Google Gemini` (Chat Model) encargado de la lógica de negocio y toma de decisiones.
* **Capa de Seguridad (Guardrails):** `Llama 3.3 (70B)` actúa como filtro para detectar intentos de manipulación o contenido inapropiado antes de procesar el mensaje.
* **Memoria:** Ventana de memoria buffer para mantener el contexto de la conversación.

---

## 🛠️ Herramientas y Sub-flujos (Tools)

El agente tiene acceso a un kit de herramientas especializadas ubicadas en `/workflows/tools/`:

### 1. 📅 ReservarCanchaFulbot

Gestiona el proceso de alta de turnos.

* **Validación:** Verifica que el horario sea "en punto" (HH:00).
* **Disponibilidad:** Llama internamente a `VerDisponibilidadFulbot`.
* **Pagos:** Si hay disponibilidad, inserta la reserva en PostgreSQL y genera un **Link de Pago de Mercado Pago** con vencimiento de 15 minutos.

### 2. 🔍 VerDisponibilidadFulbot

Lógica técnica para evitar solapamientos.

* Utiliza tipos de datos de rango (`tsrange`) en PostgreSQL para asegurar que no existan colisiones horarias entre reservas.

### 3. ⏰ PosiblesHorarios

Herramienta de asistencia al cliente.

* Si el horario solicitado está ocupado, este flujo escanea la agenda del día (09:00 a 23:00) y devuelve una lista de franjas horarias libres al cliente.

### 4. 📋 RecuperarReservasFulbot

* Permite al cliente consultar todas sus reservas activas asociadas a su nombre y número de teléfono, devolviendo un listado formateado.

### 5. ❌ EliminarReservaFulbot

Aplica las políticas comerciales automáticamente:

* **Antelación > 24hs:** Cancela la reserva y confirma el reintegro de la seña.
* **Antelación < 24hs:** Cancela el turno pero informa que la seña no es reembolsable.
* **Casos críticos:** Si hay errores, deriva automáticamente al soporte humano.

---

## ⚙️ Integraciones y Automatizaciones

El flujo principal (`Fulbot-AI-Manager.json`) incluye procesos de segundo plano:

* **💳 Webhook de Mercado Pago:** Procesa notificaciones en tiempo real, valida la firma de seguridad, acredita el pago en la base de datos y envía un mensaje de confirmación automática al cliente.
* **🔄 Sincronización con Google Sheets:** Cada cambio en la base de datos (alta, modificación o cancelación) se refleja automáticamente en una hoja de cálculo para control administrativo.
* **🧹 Limpieza Automática:** Un cronjob cada 30 minutos cancela aquellas reservas en estado "ESPERA" que no completaron el pago a tiempo.
* **⚡ Buffer de Mensajes (Redis):** Gestiona la cola de mensajes entrantes para evitar que mensajes múltiples del mismo usuario confundan a la IA.

---

## 📋 Requisitos Técnicos

1. **n8n** (Versión reciente con soporte para LangChain).
2. **PostgreSQL** con extensión `btree_gist` para manejo de rangos.
3. **Redis** para la gestión de estados y buffer.
4. **API Keys:** Google Gemini, Groq (Llama 3.3), Mercado Pago y Evolution API (WhatsApp).

## 🚀 Configuración de la Base de Datos

Es fundamental ejecutar el esquema definido en el flujo de **Creación de la DB** para habilitar la protección anti-solapamiento:

```sql
-- 1. EXTENSIONES (Nivel Base de Datos)
-- Necesaria para que funcione la restricción de exclusión con enteros (NUMEROCANCHA)
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- 2. REINICIAR ESQUEMA
DROP SCHEMA IF EXISTS negocio CASCADE;
CREATE SCHEMA negocio;

-- 3. TABLA CANCHAS (Debe crearse primero porque 'reservas' depende de ella)
CREATE TABLE IF NOT EXISTS negocio.canchas (
    "NUMERO" SERIAL PRIMARY KEY,
    "FUTBOL" INTEGER,
    "PRECIO" NUMERIC(15, 2),
    "ESTADO" VARCHAR(10)
);

-- 4. TABLA RESERVAS (Con la protección anti-solapamiento incluida)
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

```

---

*Desarrollado para la gestión inteligente de complejos deportivos.*
