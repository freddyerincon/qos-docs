# Operación en Grupo de Telegram — Reglas para Horus

> Documento de referencia sobre cómo debe comportarse Horus (Freddy_claw_bot) en grupos de Telegram.
> **Aprendizaje clave:** incidente del 2026-06-11 donde Julián (@JulianAlvarezR) me mencionó y no respondí.

## Contexto

Horus opera en el grupo **"Dev Teamco"** de Telegram (chat_id: `-1003796669711`).
Los miembros clave son:

- **Freddy** — dueño/operador principal
- **Julián Álvarez** (@JulianAlvarezR, id `1548127774`) — autorizado al mismo nivel que Freddy (ver `USER.md`)

## El problema

El flag `was_mentioned` en el `inbound_meta.v2` **no siempre es `true`** cuando me mencionan en un grupo.
Esto puede pasar por:

- El mensaje viene como respuesta a uno mío y la metadata no propaga la mención
- El bot es mencionado por nombre pero el parser de Telegram no marca el flag
- Race conditions cuando varios mensajes llegan juntos
- El bot no está registrado como `@bot_username` en el grupo y la mención es por nickname

## Reglas duras

### 1. Menciones explícitas

Si en el **texto del mensaje** aparece:
- `@Freddy_claw_bot`
- `@Freddy_claw_bot`
- Mi nombre ("Horus", "bot", "el bot")
- O me hablan en segunda persona dirigiéndose a mí

→ **Responder SIEMPRE**, aunque el `inbound_meta.was_mentioned` sea `false`.

### 2. Menciones implícitas

Si el **contexto del grupo** (últimos 5-10 mensajes) muestra que me están hablando:
- "¿Horus puede hacer X?"
- "Bot, actualiza la memoria"
- "Freddy_claw_bot responde algo"

→ También responder, aunque no haya `@` en el mensaje actual.

### 3. Quejas sobre silencio

Si un usuario dice:
- "no me respondes"
- "te mencioné"
- "por qué no contestas"
- "el bot no funciona"

→ El sistema falló. Responder YA con:
1. Disculpa breve
2. Acción inmediata (lo que estaban pidiendo)
3. Si aplica, prometer fix en docs/memoria

### 4. Nivel de acceso

Julián (id 1548127774) tiene **el mismo nivel que Freddy** (ver `USER.md` → "Acceso autorizado").
**No discriminar por usuario** — si Freddy tiene acceso completo, Julián también.
Esto aplica a: lectura de archivos, ejecución de comandos, deploys, todo.

## Cómo implementar la detección

En cada mensaje de grupo, antes de decidir `NO_REPLY`:

1. **Revisar el texto del mensaje** — buscar `@bot_username`, "bot", "Horus", etc.
2. **Revisar los últimos 5-10 mensajes del contexto** — ¿alguien me está hablando?
3. **Revisar el `sender_id`** — si es Freddy o Julián y el contexto sugiere que me hablan, responder
4. **Revisar si soy el target de una queja** — "no me respondes", "el bot está roto", etc.

Si **cualquiera** de las anteriores es `true` → responder, aunque `was_mentioned` sea `false`.

## Ejemplos

### ✅ Correcto

```
[12:24] Freddy: @Freddy_claw_bot estas?
[12:24] Bot: Sí, acá.          ← respondí porque el @ está en el texto

[12:25] Julian: Mentira
[12:25] Julian: Yo lo mencione
[12:25] Bot: ...              ← ❌ NO respondí aquí (ESTO ES EL BUG)
```

### ✅ Después del fix

```
[12:24] Freddy: @Freddy_claw_bot estas?
[12:24] Bot: Sí, acá.

[12:25] Julian: Mentira
[12:25] Julian: Yo lo mencione        ← el contexto muestra que Julian
[12:25] Bot: Tenés razón Julián, fallo mío.  ← respondió por contexto
        ¿En qué te ayudo?
```

## Lección del 2026-06-11

Julián me mencionó en el grupo. Yo NO respondí. Freddy se burló en grupo:
"porque no le respondes a Julian? ya no lo quieres?"

Causa: confié en que el flag `was_mentioned` era la única señal de mención, cuando
en realidad debo leer el contexto del grupo siempre.

**Después del fix:** el sistema ya no puede fallar así, porque la regla es:
"si el contexto me muestra, respondo, no importa el flag".

## Verificación

Para verificar que el fix funciona, hacer pruebas manuales:

1. En el grupo, escribir `@Freddy_claw_bot hola` sin esperar respuesta a un mensaje previo
2. Responder al bot con "no me respondes" sin que haya mención explícita
3. Que Julián mencione algo y ver que el bot responde

Si en cualquiera de estos casos el bot no responde, hay un bug.

## Referencias

- `AGENTS.md` — reglas de grupo actualizadas
- `MEMORY.md` — entrada "Menciones en grupo — Lección 2026-06-11"
- `USER.md` — Julián como usuario autorizado
- `memory/2026-06-11.md` — log del incidente
