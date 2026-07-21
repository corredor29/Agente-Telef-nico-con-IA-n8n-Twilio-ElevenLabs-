# Documentación: Agente Telefónico con IA (n8n + Twilio + ElevenLabs)

## 1. Descripción general del proyecto

El objetivo es construir un sistema que conteste llamadas telefónicas de forma automática, manteniendo una **conversación fluida y natural** con el cliente (sin menús tipo "marque 1 para..."), capaz de recolectar y guardar información de cada llamada, y con soporte para **diferentes voces y tonos** según el caso de uso.

### Principio de arquitectura clave

**n8n no procesa audio en tiempo real.** Sus workflows se disparan por eventos/webhooks, no manejan streams continuos de voz con baja latencia. Por eso el sistema se divide en dos capas:

- **Motor de voz conversacional** (ElevenLabs Conversational AI): lleva la conversación en vivo — escucha, entiende, decide qué responder y habla, todo con procesamiento de voz en tiempo real de baja latencia.
- **n8n**: actúa como "trastienda" — el agente de voz le pide datos o acciones (agendar, consultar, guardar) mediante webhooks cuando los necesita durante la llamada, y recibe el resumen completo al finalizar.

---

## 2. Arquitectura y flujo del sistema

```
Cliente llama → Twilio (número telefónico)
              → ElevenLabs Conversational AI Agent (STT + LLM + TTS en tiempo real)
                   ↳ durante la llamada, si necesita datos externos:
                        → Webhook → n8n (Workflow "Tools") → responde JSON → el agente sigue hablando
                   ↳ al terminar la llamada:
                        → Webhook → n8n (Workflow "Post-call") → guarda transcripción, duración, resumen
```

### Workflow 1 — "Tools" (durante la llamada, tiempo real)
- **Webhook Trigger (POST)** configurado como Server Tool del agente en ElevenLabs, en modo "Respond to Webhook".
- **Switch/IF** para bifurcar según qué herramienta pidió el agente (agenda, CRM, base de datos, etc.).
- **Nodos de lógica de negocio**: Google Calendar, Google Sheets, HTTP Request a sistemas propios, CRM.
- **Respond to Webhook**: regresa un JSON corto y rápido para que el agente continúe la conversación sin pausas largas.
- Requiere `idempotency_key`/`correlation_id` para evitar duplicados si hay reintentos de red.

### Workflow 2 — "Post-call" (al terminar la llamada)
- **Webhook Trigger**: recibe transcripción completa, duración, costo, resultado de la llamada.
- **Set/Edit Fields**: extrae los datos relevantes.
- **(Opcional) Nodo de IA**: resume o clasifica sentimiento/intención de la llamada.
- **Guardado**: Google Sheets / Airtable / base de datos / CRM.
- **(Opcional) Notificación**: Slack, WhatsApp, email.

### Workflow 3 — Administración
- Gestión de prompts del agente, cambios de voz, actualización de base de conocimiento (RAG). No corre por llamada, es de uso interno.

---

## 3. Tecnologías y por qué se eligieron

| Tecnología | Rol en el proyecto | Por qué esta y no otra |
|---|---|---|
| **Twilio** | Infraestructura telefónica (recibir/hacer llamadas reales) | Estándar de la industria, integración nativa con ElevenLabs sin configuración manual de TwiML, cobertura en 180+ países |
| **ElevenLabs Conversational AI** | Motor de voz en tiempo real (STT + LLM orquestado + TTS) | Respuestas de menos de un segundo, soporta 31+ idiomas, mejor naturalidad de voz del mercado, integración nativa con Twilio y n8n |
| **n8n** | Orquestador de lógica de negocio y almacenamiento de datos por llamada | Visual, rápido de mantener, se conecta a cientos de sistemas (CRM, Sheets, bases de datos) sin escribir un backend desde cero |
| **PostgreSQL** (para producción) | Base de datos de n8n | SQLite (la opción default) bloquea el archivo completo al escribir; con tráfico concurrente de llamadas se generan conflictos |
| **Redis** (para producción con volumen) | Cola de ejecución de n8n (queue mode) | Necesario para manejar varias llamadas simultáneas sin que los webhooks se encolen y generen latencia |
| **Docker + Docker Compose** | Despliegue de n8n | Estándar de facto, actualizaciones simples, entornos reproducibles |

### Alternativas open source (para bajar costo variable, con trade-offs)
- **Whisper** (STT local) en vez de Deepgram/ElevenLabs Scribe — gratis, pero requiere servidor propio.
- **Ollama + modelo local** (Llama, Mistral) en vez de OpenAI/Claude — gratis en uso, pero requiere GPU y pierde velocidad/calidad de razonamiento frente a APIs comerciales.
- **Piper / Coqui TTS** en vez de ElevenLabs — voces gratis, pero notablemente menos naturales; no recomendado si el requisito central es "que se note una conversación fluida".
- La parte de **telefonía (Twilio/Telnyx/Plivo) siempre tiene costo**, no existe alternativa gratuita real a escala.

---

## 4. Voces y tonos

ElevenLabs permite:
- Elegir entre una librería amplia de voces predefinidas con distintos tonos (formal, cálido, enérgico, calmado, etc.).
- Clonar o diseñar una voz personalizada (branding propio).
- Asignar voces distintas por agente/caso de uso (ej. un agente de ventas con tono entusiasta, uno de soporte con tono calmado).
- Elegir entre modelo **Multilingual v2** (mejor calidad, más lento, para audio de producción) o **Flash** (menor latencia, ideal para conversación en tiempo real).

---

## 5. Tiempo y dificultad del proyecto

*Estimación basada en un desarrollador con nivel avanzado en n8n/APIs/webhooks, dedicando 8 horas diarias.*

| Fase | Horas estimadas | Dificultad | Notas |
|---|---|---|---|
| Setup de cuentas e infraestructura | 2–4h | Baja | Twilio, ElevenLabs, n8n, conexión entre ambos |
| Diseño y prompt del agente conversacional | 4–8h | Media | Requiere iteración; "sonar natural" es subjetivo, se ajusta probando |
| Workflow 1 — Tools (lógica en vivo) | 6–12h | Media | Depende del número de herramientas necesarias |
| Workflow 2 — Post-call (logging) | 3–5h | Baja | Procesamiento sin presión de tiempo real |
| Pruebas end-to-end + ajuste de latencia | 6–10h | Media-Alta | Fase más subestimada; requiere llamadas reales repetidas |
| Seguridad, errores, edge cases | 3–6h | Media | Autenticación de webhooks, manejo de fallos, reintentos |

### Totales
- **MVP funcional / demo presentable:** 25–40 horas → **3 a 5 días** (8h/día)
- **Listo para vender a clientes reales:** 45–70 horas → **6 a 9 días** (8h/día)

**Recomendación para presentar fecha:** comunicar **1 semana para el MVP** y **2 semanas para versión vendible**, dejando margen para imprevistos (verificación de número Twilio, aprobaciones, ajuste de latencia).

### Dónde está la dificultad real
No es el código en sí — es:
1. Afinar que la conversación se sienta natural (prompt engineering + configuración de voz).
2. Controlar la latencia en producción.
3. Decidir correctamente qué vive en el agente (conocimiento estático) vs qué vive en n8n (acciones dinámicas).

---

## 6. Solución al problema de latencia

| Fuente de latencia | Solución |
|---|---|
| TTS/LLM del agente | Usar modelo **Flash** de ElevenLabs (menor latencia que Multilingual v2) y un LLM rápido, no de razonamiento extendido |
| Detección de turnos | Ajustar bien el VAD (ni muy sensible ni muy relajado) |
| Webhook → n8n | Minimizar nodos en la cadena crítica; evitar llamadas externas encadenadas en secuencia; paralelizar cuando sea posible |
| Datos repetitivos | No consultarlos por webhook cada vez — meterlos como conocimiento estático en el prompt del agente (RAG) |
| Hosting de n8n | Servidor bien dimensionado y geográficamente cercano al de ElevenLabs; evitar instancias compartidas lentas |
| Concurrencia | Queue mode con Redis si hay varias llamadas simultáneas |
| Percepción del usuario | Configurar frases de relleno ("dame un segundo que reviso eso") mientras corre un tool que tarda |

---

## 7. Instalación y stack técnico (versiones recomendadas, julio 2026)

| Componente | Versión / detalle |
|---|---|
| Docker + Docker Compose | Última estable |
| Node.js (solo si se instala n8n vía npm) | v24 (LTS activo recomendado para proyectos nuevos; v22 sigue soportado hasta abril 2027) |
| n8n | Última versión estable, imagen oficial `n8nio/n8n` |
| PostgreSQL | 15+ |
| Redis | Última estable (si se usa queue mode) |
| Reverse proxy + SSL | Nginx o Caddy + Let's Encrypt (obligatorio, el webhook debe ser HTTPS) |
| VPS (pruebas) | mínimo 2 vCPU / 2GB RAM |
| VPS (producción) | recomendado 4 vCPU / 4GB+ RAM (sube si se procesan contextos de IA grandes) |

---

## 8. Recursos y costos estimados

### Costos fijos mensuales (infraestructura)
| Ítem | Costo aproximado |
|---|---|
| Número Twilio (local US) | ~$1.15/mes |
| VPS producción (4 vCPU/4GB+) | variable según proveedor, desde ~$20–40/mes |
| Plan ElevenLabs (Pro, recomendado para producción) | ~$99/mes (1,238 min incluidos) |

### Costos variables por uso
| Ítem | Costo aproximado |
|---|---|
| Twilio inbound (número local) | ~$0.0085/min |
| Twilio outbound (US) | ~$0.013–0.014/min |
| ElevenLabs — minutos extra fuera del plan | ~$0.08/min (doble si se excede concurrencia) |
| LLM (aparte del plan de ElevenLabs) | según proveedor/modelo elegido |

### Costo total aproximado por minuto de llamada
**$0.08 – $0.15 por minuto**, sumando Twilio + ElevenLabs + LLM. Esta cifra es la base para definir el precio de venta a clientes (por minuto, por llamada, o suscripción).

### Nota sobre licencia de n8n
La Sustainable Use License (gratuita) cubre uso interno, incluyendo agencias que operan el servicio para clientes sin exponerles n8n directamente. Si el modelo de negocio evoluciona a exponer workflows a clientes o convertir n8n en el motor de un SaaS que se vende, se requiere una licencia Enterprise o Embed — se recomienda confirmar el caso específico con **license@n8n.io** antes de escalar comercialmente.

---

## 9. Riesgos del proyecto

**Técnicos**
- Usar SQLite en producción (bloquea con tráfico concurrente) — migrar a PostgreSQL desde el inicio.
- Latencia que funciona con 1 llamada de prueba pero falla con varias simultáneas si el servidor no está bien dimensionado.
- Pérdida de `N8N_ENCRYPTION_KEY` → pérdida de acceso a todas las credenciales guardadas.
- Límite de llamadas concurrentes según plan de ElevenLabs (Free: 4, Starter: 6, Creator: 10, Pro: 20...); exceso se cobra a tarifa doble.

**De negocio / legal**
- Confirmar con n8n si el modelo de negocio requiere licencia comercial.
- Verificación de número Twilio puede tardar días según país/tipo de número — solicitar cuanto antes.
- Revisar normativa local sobre grabación de llamadas y consentimiento del cliente.

**De producto**
- Manejar expectativas: siempre habrá casos de malinterpretación (acentos, ruido, interrupciones). Se recomienda un mecanismo de transferencia a humano como plan B.

---

## 10. Resumen ejecutivo

| Aspecto | Resumen |
|---|---|
| Stack principal | Twilio + ElevenLabs Conversational AI + n8n |
| Tiempo MVP | 3–5 días (8h/día, nivel avanzado) |
| Tiempo versión vendible | 6–9 días |
| Costo por minuto de llamada | $0.08–$0.15 |
| Costo fijo mensual mínimo | ~$120–150/mes (VPS + plan ElevenLabs Pro + número Twilio) |
| Mayor riesgo técnico | Latencia en producción con múltiples llamadas simultáneas |
| Mayor riesgo de negocio | Licencia de n8n si se expone a clientes finales |
