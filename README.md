# local-ai-telegram-agent

Agente de IA que corre en mi PC con un modelo local y al que le hablo desde Telegram.

Lo monté porque quería probar a tener un asistente propio sin depender de APIs de pago y aprender cómo funcionan los LLMs en local: cuánta memoria necesitan, cómo se cuantizan y cómo un agente usa herramientas.

## Cómo está montado

- **LM Studio** sirve el modelo en `http://localhost:1234/v1` (API compatible con OpenAI).
- **Modelo:** Gemma 4 26B A4B QAT. Antes probé el Gemma 4 12B, pero el 26B usa mejor las herramientas.
- **GPU:** AMD Radeon RX 9070 XT (16 GB) con el runtime ROCm de LM Studio.
- **Agente:** [Hermes Agent](https://github.com/NousResearch/hermes-agent) de Nous Research, conectado a LM Studio como proveedor personalizado.
- **Telegram:** el gateway de Hermes conecta el agente con un bot creado con @BotFather.

## Qué hace

- Responder preguntas desde Telegram.
- Ejecutar comandos en el PC. Tengo las aprobaciones en modo manual, así que me pide confirmación antes de ejecutar nada.
- Buscar información en la web.
- Transcribir notas de voz con Whisper en local.
- Recordar cosas entre conversaciones.

## Cosas que aprendí

- El 26B A4B es un modelo *Mixture of Experts*: tiene 26B parámetros pero solo usa unos 4B por token, por eso va bastante rápido.
- Con 64k de contexto no me cabía todo en la VRAM, así que dejé parte de los expertos en la CPU.
- Un agente que puede ejecutar comandos desde el móvil es peligroso si cualquiera puede escribir al bot. Por eso las aprobaciones son manuales y el token va en un `.env` que no se sube al repo.

## Configuración

En [`config/config.example.yaml`](config/config.example.yaml) está la parte de la configuración de Hermes que he tocado (sin claves), y en [`config/.env.example`](config/.env.example) las variables que hacen falta.

Para arrancarlo:

1. Cargar el modelo en LM Studio y arrancar el servidor local.
2. Poner el token del bot en el `.env` de Hermes.
3. Ejecutar `hermes gateway run`.

## Pendiente

- Añadir capturas.
- Hacer un script para arrancar todo de una vez.
