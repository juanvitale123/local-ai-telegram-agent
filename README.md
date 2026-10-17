# Local AI Agent on Telegram

Agente de IA autónomo que funciona **100 % en mi propio hardware** y que controlo desde Telegram. Un LLM de 26B parámetros servido en local con LM Studio sobre GPU AMD, orquestado por [Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research), con herramientas para ejecutar comandos en el PC, buscar en la web y recordar contexto entre conversaciones.

Sin APIs de pago, sin enviar los datos a terceros y con aprobación manual de cualquier acción sobre el sistema.

## Qué puede hacer

| Capacidad | Cómo funciona |
|---|---|
| Asistente conversacional | Responde desde Telegram usando el modelo local |
| Comandos en el PC | Ejecuta comandos en una shell persistente de Windows, **siempre con aprobación manual** |
| Búsqueda web | Busca y extrae información de páginas para responder con datos actuales |
| Notas de voz | Transcribe los audios de Telegram en local con Whisper (STT) |
| Memoria | Recuerda preferencias y contexto entre sesiones |
| Contextos largos | Compresión automática del historial al llegar al 50 % de la ventana |

## Arquitectura

```
 ┌──────────────┐        ┌─────────────────────────── PC Windows ───────────────────────────┐
 │   Telegram   │        │                                                                   │
 │  (móvil/PC)  │◄──────►│  Hermes Gateway ──► Hermes Agent ──► LM Studio (API OpenAI-compat.)│
 └──────────────┘  Bot   │        │                 │            http://localhost:1234/v1     │
                   API   │        │                 │                    │                     │
                         │        │                 ├── Terminal (local, aprobación manual)    │
                         │        │                 ├── Web search / extract                   │
                         │        │                 ├── Memoria persistente                    │
                         │        │                 └── STT local (Whisper)                    │
                         │        │                                      ▼                     │
                         │        │                    Gemma 4 26B A4B (MoE, QAT) en GPU AMD   │
                         │        │                    ROCm · 64k contexto · expertos en CPU    │
                         └────────┴─────────────────────────────────────────────────────────────┘
```

## Stack

| Capa | Tecnología |
|---|---|
| Hardware | AMD Ryzen 7 7800X3D · AMD Radeon RX 9070 XT (16 GB VRAM) |
| Servidor de inferencia | LM Studio con runtime ROCm, endpoint compatible con OpenAI |
| Modelo | Google Gemma 4 26B A4B QAT (Mixture of Experts, ~4B parámetros activos) |
| Agente | Hermes Agent v0.22 (Nous Research) |
| Canal | Telegram Bot API a través del gateway de Hermes |
| Voz | Whisper `base` en local |
| SO | Windows 11 |

## Decisiones técnicas

**¿Por qué un modelo MoE de 26B y no uno denso de 12B?**
Gemma 4 26B A4B solo activa ~4B parámetros por token, así que genera casi tan rápido como un modelo pequeño pero con la calidad de uno grande. Probé primero Gemma 4 12B QAT y el 26B A4B razona mejor al usar herramientas.

**¿Cómo cabe en 16 GB de VRAM con 64k de contexto?**
- Cuantización QAT (Quantization-Aware Training): pierde mucha menos calidad que una cuantización posterior.
- Parte de las capas de expertos se descargan a la CPU/RAM, dejando VRAM libre para la caché KV del contexto largo.
- Compresión del historial del agente al 50 % de la ventana, conservando los primeros y los últimos mensajes.

**¿Por qué AMD + ROCm?**
Es el hardware que tengo. LM Studio soporta ROCm en Windows, lo que evita depender de CUDA.

## Seguridad

Un agente que puede ejecutar comandos en tu PC desde el móvil es un riesgo si no se controla. Medidas aplicadas:

| Riesgo | Medida |
|---|---|
| Alguien escribe al bot | El bot solo atiende a usuarios emparejados/autorizados |
| El modelo ejecuta algo destructivo | `approvals.mode: manual`: cada comando requiere confirmación |
| Tareas programadas sin supervisión | `cron_mode: deny`: no se ejecutan comandos desde cron sin aprobación |
| Fuga de credenciales | Tokens en `.env` (excluido de Git); redacción de secretos activada |
| Acceso a la red interna | `allow_private_urls: false` en el navegador del agente |

## Puesta en marcha

1. **LM Studio**: descargar `google/gemma-4-26b-a4b-qat`, seleccionar el runtime ROCm, contexto 64k y arrancar el servidor local (puerto 1234).
2. **Hermes Agent**: instalar en Windows y configurar el proveedor personalizado (ver [`config/config.example.yaml`](config/config.example.yaml)).
3. **Bot de Telegram**: crear el bot con @BotFather y guardar el token en `.env` (ver [`config/.env.example`](config/.env.example)).
4. Arrancar el gateway:

```powershell
hermes gateway run
```

5. Emparejar tu usuario de Telegram con el bot y probar con un mensaje.

## Estructura

```
.
├── config/
│   ├── config.example.yaml   # Extracto de la configuración relevante (sin secretos)
│   └── .env.example          # Variables necesarias, sin valores
├── docs/
│   └── screenshots/          # Capturas del bot en funcionamiento
└── README.md
```

## Próximos pasos

- Integración con Home Assistant (plugin ya habilitado)
- Despliegue del gateway como servicio de Windows para arranque automático
- Monitorización del uso de GPU con Prometheus + Grafana
