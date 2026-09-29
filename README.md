# Job Intelligence & ATS Pipeline

## 1. Qué es
Sistema ATS automatizado, desacoplado y self-hosted que ingiere vacantes multititular de TI, deduplica relacionalmente y ejecuta un pre-filtrado determinístico en RAM ($0 en tokens) antes de aplicar inferencia semántica con IA.

## 2. Arquitectura
```text
[ Fuentes: GetOnBrd, DonWeb, Compromiso, Otros ]
                         │
                         ▼
             [ Adaptadores Canónicos ]
                         │
                         ▼
        [ Deduplicación: PostgreSQL 16 ]
                         │
               (is_duplicate == false)
                         │
                         ▼
  [ Pre-Filtrado JS (RAM / $0 Tokens): Modalidad & Perfil ]
                         │
             (_filtro_perfil_ok == true)
                         │
                         ▼
      [ Inferencia IA: Groq / Gemini Flash ]
                         │
                         ▼
    [ Persistencia Final DB + Bot Telegram ]
```

## 3. Stack
* **Orquestación & Workflows:** n8n Self-Hosted.
* **Infraestructura:** VPS Linux Cloud Provider, Docker Containers.
* **Base de Datos Relacional:** PostgreSQL 16 (JSONB, Unique Constraints, B-Tree Indexes).
* **Lenguajes & Runtimes:** JavaScript (Node.js ES6+ V8), SQL DDL/DML.
* **Modelos de IA / Inferencia:** Groq API (openai/gpt-oss-20b); Gemini Flash como respaldo planificado.

## 4. Incidentes / decisiones

### Incidente 1: Evaluación asíncrona e inversión del pipeline
* **Síntoma:** El nodo evaluador `IF` enviaba el 100% de las vacantes a la rama `FALSE` (0 vacantes en `TRUE`).
* **Causa:** El nodo `IF` evaluaba la variable `_filtro_modalidad_ok` antes de que el nodo de código JavaScript la generara en `$json` (resultando en `undefined`).
* **Solución:** Reestructuración secuencial estricta: Ingesta ──► Deduplicación DB ──► Code Node (Inyección Banderas) ──► IF Evaluator Node.

### Incidente 2: Fallo de sintaxis en V8 Task-Runner (n8n v2.10.2)
* **Síntoma:** Error `SyntaxError: Invalid or unexpected token` al ejecutar el script en el entorno self-hosted.
* **Causa:** Filtrado involuntario de un carácter de escape (`\`) en la llamada `$input.all()` (`\$input.all()`).
* **Solución:** Corrección sintáctica explícita (`const items = $input.all();`) operando en modo `Run Once for All Items`.

### Incidente 3: Inyección SQL y desborde por comillas simples
* **Síntoma:** Fallos de sintaxis en sentencias `INSERT` al procesar vacantes con apóstrofes o comillas simples en la descripción.
* **Causa:** Interpolación directa de cadenas sin higienizar en la consulta SQL de PostgreSQL.
* **Solución:** Higienización en el middleware (`.replace(/'/g, "''")`) e inserción idempotente con `ON CONFLICT (source_portal, external_id) DO UPDATE`.

## 5. Estado real

### En Producción (Implementado y Funcional)
* **Ingesta & Adaptadores:** Conexión y normalización de GetOnBrd (API REST) y DonWeb / Compromiso Careers (Zoho Recruit API).
* **Deduplicación:** Motor de verificación relacional idempotente en PostgreSQL 16 mediante `SELECT EXISTS(...)`.
* **Pre-Filtrado Determinístico (FinOps $0 Tokens):** Filtro 1 (Modalidad 100% Remoto) y Filtro 2 (Perfil Sysadmin/DevOps/Infra vs Stop-Rules) ejecutados en RAM.
* **Pre-Filtrado Determinístico: * **Inferencia Semántica:** Groq API con sanitización de payload y throttling; persistencia idempotente.

### En Plan (Roadmap & Fases Futuras)
* **Adaptadores Pendientes:** Extractor e integrador para SAP SuccessFactors.
* **Fallback de inferencia:** conmutación automática de Groq a Gemini Flash ante errores 429/500. Conexión del nodo `HTTP Request` para Groq / Gemini Flash con JSON Schema estricto.
* **Interfaz & Notificaciones:** Bot interactivo de Telegram con botones de postulación (`[🎯 Postular]`, `[💾 Guardar]`) y Webhook de callback para marcar `applied = true` en PostgreSQL.
