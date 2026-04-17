# Comunicación con Spruce y la solución del Archive Sweeper (MLID-2167)

> Documento de análisis derivado de una conversación de diseño. Resume cómo funciona la integración con Spruce, qué webhooks recibimos, el problema de race condition que motivó el ticket MLID-2167, y por qué un rediseño de la secuencia lógica no elimina la necesidad del sweeper.

---

## 1. ¿Recibimos webhook de Spruce cuando un mensaje es archivado?

**No.** Spruce no envía un webhook cuando una conversación es archivada. Los webhooks que recibimos de Spruce son eventos de mensaje / conversación entrantes (los que disparan *nuestro* flujo de archivado). No hay un callback de "conversation archived" al que estemos suscritos ni que manejemos.

La confirmación de que el archivado ocurrió viene de la **respuesta HTTP síncrona** a nuestro propio `PATCH` contra el endpoint `updateConversation` de Spruce — no de un callback.

### ¿Qué pasa cuando el sweeper le pide a Spruce archivar un texto?

`spruceArchiveSweeper.ts` → `noteAndArchiveConversation()` → `archiveSpruceConversation()` (`apps/web/services/integrations/spruce.ts:781`):

1. **Reclamo atómico** — `findOneAndUpdate` con clave en `spruce.archivedAt: { $exists: false }`. Estampa `archivedAt` por adelantado para que un webhook tardío no procese dos veces.
2. **Posteo de nota interna** — `postSpruceInternalNote(conversationId, 'Processed by LISA')`.
3. **Espera 2 s** para que la nota aterrice antes de cerrar la conversación.
4. **Llamada de archivado** — `api.updateConversation({ archived: true }, { conversationId })`. Spruce responde inline con el objeto de conversación actualizado (`response.data.conversation`). Ese HTTP 2xx **es** nuestra confirmación de archivado.
5. **En éxito** → `noteAndArchiveConversation` retorna `true`, `archivedAt` queda estampado, el sweeper sigue.
6. **En fallo (excepción)** → el sweeper hace `$unset` de `archivedAt` para que el siguiente tick (5 min) reintente.

El loop con Spruce es completamente request/response. La protección `archivedAt` en el handler del webhook entrante existe para hacer idempotente la **reentrega del mensaje original por parte de Spruce** — no para consumir un evento de "archived".

---

## 2. ¿Qué webhooks recibimos de Spruce y qué los dispara?

La suscripción se registra sin filtro de eventos (`POST /api/admin/integrations/spruce/subscription` → `createWebhookEndpoint({ name, url })`), así que **Spruce entrega todos los tipos de evento a un solo endpoint**: `POST /api/webhooks/spruce_conversations`. Nuestro handler `handleSpruceConversation` (`apps/web/services/webhooks/handlers/spruce_conversation.ts:209-391`) discrimina por `body.type` y solo actúa sobre cuatro tipos — el resto se descarta silenciosamente (`else { return; }`).

### Eventos que sí procesamos

| `type` | Disparado por (lado Spruce) | Qué hacemos |
|---|---|---|
| `conversation.created` | Se abre una nueva conversación en Spruce (ej. primer intercambio SMS con un número) | Upsert en nuestra colección `Conversation` con clave `conversation_id`, guardamos el payload completo en `data` |
| `conversation.updated` | Cualquier cambio en una conversación (asignado, tags, flag de archivado, etc.) | Mismo upsert — el payload sobrescribe `data` |
| `conversationItem.created` | Se añade un nuevo ítem a una conversación: SMS entrante, SMS saliente que enviamos, fax entrante, nota interna, etc. | Upsert en `ConversationItem`. **Dos ramas con efectos secundarios:** ① si `direction='outbound'` + `requestId` → enlazamos `spruce.conversationId`/`appUrl` al `Message` original y corremos el flujo de archivado de SMS; ② si `direction='inbound'` con adjuntos de tipo documento → agendamos `spruceIntakeJob` para ingresar el fax/PDF |
| `conversationItem.updated` | Edits a un ítem existente (raro; ej. cambios de status en delivery saliente) | Igual que `created` — pasa por el mismo upsert y ramas de efectos |

### Eventos que **NO** procesamos

Cualquier otra cosa que Spruce emita (delivery receipts, cambios de contacto/endpoint, membresía de equipo, **cambios de estado de archivado**, etc.) llega a nuestro endpoint, falla el check de `type` y se descarta. **No hay señal entrante que nos diga "Spruce archivó esta conversación"** — por eso MLID-2167 tuvo que apoyarse en la respuesta síncrona de `updateConversation({ archived: true })` más el sweeper como red de seguridad.

### Bonus: endpoint legacy

`apps/web/pages/api/webhooks/spruce.ts` existe pero es código muerto — verifica la firma y luego retorna un `{ success: true, jobId: '123' }` hardcodeado sin despachar nada. Todo el tráfico real va por `spruce_conversations.ts`.

---

## 3. Cómo funciona la comunicación con Spruce — alto nivel

Imagina que **Spruce es como WhatsApp pero para clínicas**: es la herramienta que el equipo usa para mandar y recibir mensajes de texto (SMS) con los pacientes. Nuestra app (LISA) habla con Spruce todo el día para automatizar tareas que antes una persona tenía que hacer a mano.

La comunicación va en **dos direcciones**, como una conversación telefónica.

### 3.1 Nosotros le hablamos a Spruce (peticiones salientes)

Cuando LISA quiere que Spruce haga algo, le manda una petición HTTP — es como hacer una llamada telefónica:

- **"Mándale este SMS al paciente"** → cuando se acerca una cita, LISA le pide a Spruce que envíe el recordatorio.
- **"Pon esta nota interna en la conversación"** → para que el equipo vea que LISA ya procesó el mensaje (la famosa nota *"Processed by LISA"*).
- **"Archiva esta conversación"** → para que desaparezca de la bandeja de entrada del equipo y no la tengan que tocar.

En cada caso Spruce contesta inmediatamente con un *"listo, hecho"* o *"falló por X razón"*. Esa respuesta es nuestra única confirmación — no hay un mensajero que venga después a avisarnos.

### 3.2 Spruce nos habla a nosotros (webhooks)

Aquí es donde aparecen los **webhooks**. Un webhook es básicamente un timbre: cada vez que pasa algo en Spruce, Spruce toca el timbre de LISA mandándole un paquetito de información.

Tenemos **un solo timbre instalado** (`/api/webhooks/spruce_conversations`), y Spruce nos avisa de **todo** lo que pasa por ahí. Pero LISA solo le presta atención a cuatro tipos de aviso (ver tabla en sección 2).

### 3.3 El "aviso de mensaje" es el que hace todo el trabajo

Cuando llega el aviso de "hay un mensaje nuevo", LISA mira de qué tipo es:

- **¿Es un SMS que nosotros enviamos?** → Spruce nos devuelve el ID de la conversación que se acaba de crear. Lo guardamos en nuestro mensaje y luego le pedimos a Spruce que ponga la nota *"Processed by LISA"* y archive la conversación. Todo automático.
- **¿Es un fax o documento que llegó del paciente?** → Lanzamos un trabajo en segundo plano (`spruceIntakeJob`) que descarga el PDF, lo procesa y lo mete en el sistema del paciente.

### 3.4 Lo que Spruce **NO** nos avisa

Spruce no nos avisa cuando una conversación se archiva. O sea: cuando le decimos *"archiva esto"*, su respuesta inmediata (*"OK, archivado"*) es la única señal que tenemos. No hay un segundo mensajero que venga después a confirmarlo.

Por eso en el ticket MLID-2167 tuvimos que añadir el **sweeper** (un proceso que cada 5 minutos revisa si hay mensajes que se quedaron sin archivar) — porque si la respuesta inmediata se pierde por cualquier razón, no tenemos forma de enterarnos por webhook. Toca preguntarle a Spruce nosotros mismos.

### Resumen en una frase

**LISA habla con Spruce por HTTP (peticiones que tienen respuesta inmediata), y Spruce habla con LISA por webhooks (avisos que llegan solos cuando pasa algo)** — pero los webhooks solo cubren mensajes y conversaciones nuevas, no eventos como "se archivó".

---

## 4. El problema original: la race condition del MLID-2167

### 4.1 El escenario "feliz" (cómo *debería* funcionar)

Cuando LISA manda un SMS de recordatorio a un paciente, pasan estas cosas en orden:

1. **LISA → Spruce:** *"Manda este SMS. Toma este `requestId` para que lo identifiquemos después."*
2. **LISA guarda en su base de datos:** *"Mensaje enviado, requestId = ABC123."*
3. **Spruce envía el SMS al paciente.**
4. **Spruce → LISA (webhook):** *"Oye, acabo de crear la conversación XYZ y el `requestId` es ABC123."*
5. **LISA recibe el webhook**, busca en su base el mensaje con `requestId = ABC123`, le pega encima el `conversationId = XYZ`, y luego le dice a Spruce: *"Pon la nota 'Processed by LISA' y archiva la conversación XYZ."*

Todo bonito. El paciente recibe el SMS, el equipo no ve el mensaje en su bandeja porque ya está archivado, y nadie tiene que tocar nada manualmente.

### 4.2 El problema: los pasos 2 y 4 corren una carrera

Aquí está la trampa. Imagina que el paso 2 (*"LISA guarda en su base de datos"*) **es lento**. Quizás la base de datos está ocupada, quizás hay una llamada externa antes, quizás el guardado se hizo "diferido" para después.

Y resulta que Spruce es **rapidísimo**. A veces el webhook del paso 4 llega **antes** de que LISA termine el paso 2.

Entonces lo que pasa es:

1. LISA → Spruce: *"manda el SMS, requestId = ABC123."*
2. (LISA empieza a guardar, pero todavía no termina)
3. Spruce manda el SMS.
4. **Spruce → LISA (webhook): *"requestId = ABC123, conversationId = XYZ."***
5. LISA busca en su base un mensaje con `requestId = ABC123`...
6. **No encuentra nada.** Porque el paso 2 todavía no terminó.
7. El webhook se va sin hacer nada. La conversación nunca se archiva.
8. (Un instante después) LISA por fin guarda el mensaje. Pero ya nadie va a venir a buscarlo — el webhook ya pasó.

**Resultado:** la conversación se queda sin archivar para siempre. El equipo la ve en su bandeja, asume que tienen que hacer algo, pierden tiempo. Multiplica eso por cientos de mensajes al día y tienes una bandeja saturada de basura.

Esto es una **race condition** clásica: dos cosas corriendo en paralelo donde el orden de llegada no está garantizado, y el sistema solo funciona si llegan en el orden "correcto".

### 4.3 El primer fix: guardar inmediatamente (commit `a5d014cb`)

La primera solución fue obvia: **guarda el `requestId` en la base de datos antes de mandarle nada a Spruce.** Ya no hay carrera porque cuando el webhook llega, el mensaje siempre ya existe.

Esto cierra **el ~99% de los casos**. Pero quedaban tres agujeros más pequeños.

### 4.4 Los agujeros que quedaban

**Agujero 1: La microventana entre "Spruce me contestó OK" y "yo guardé en mi base"**

Aún guardando "antes", existe un parpadeo de milisegundos donde Spruce ya respondió pero el `INSERT` en Mongo todavía no terminó. Si el webhook viene en ese parpadeo, mismo problema.

**Agujero 2: Spruce reentrega el mismo webhook**

Spruce, como buen sistema, a veces manda el mismo webhook dos veces (por si la primera vez se perdió). Sin protección, LISA procesaría las dos veces — pondría **dos** notas *"Processed by LISA"* y trataría de archivar dos veces una conversación ya archivada.

**Agujero 3: Algo simplemente se cae**

- El proceso de LISA crashea justo después de mandar el SMS.
- Mongo tiene un error transitorio justo cuando guardamos.
- Spruce tiene un fallo de entrega y el webhook nunca llega.

En cualquiera de esos casos, **no hay forma de recuperarse**. El mensaje queda huérfano para siempre.

### 4.5 Por qué necesitamos el sweeper

El webhook es el camino "en tiempo real" — funciona casi siempre, es rápido, y debería seguir siendo el principal. **Pero "casi siempre" no es suficiente** para un sistema de salud donde una bandeja sucia se traduce en tiempo perdido del equipo y mensajes sin atender.

El **archive sweeper** es la **red de seguridad**. Es un trabajo en segundo plano que se ejecuta **cada 5 minutos** y hace una pregunta muy simple a Mongo:

> *"¿Hay mensajes salientes de la última hora que YA tienen un `conversationId` (o sea, el webhook sí llegó y los identificó), pero NO tienen marca de archivado? Si sí, archívalos."*

Con eso cubrimos los tres agujeros:

- **Agujero 1 (microventana):** si el webhook se pierde por timing, el sweeper lo recoge en la siguiente pasada (máximo 5 minutos de retraso).
- **Agujero 2 (reentrega):** ahora marcamos `spruce.archivedAt` cuando archivamos. Si el webhook viene dos veces, el segundo intento ve la marca y se va sin hacer nada. Misma protección sirve para que el sweeper y un webhook tardío no choquen entre sí.
- **Agujero 3 (caídas):** el sweeper no depende del webhook ni del proceso original. Aunque LISA se caiga, cuando vuelva a levantarse el sweeper revisará el último rato y arreglará todo lo que quedó pendiente.

### 4.6 La filosofía del fix

El patrón es muy clásico en sistemas distribuidos:

- **Camino primario en tiempo real (webhook):** rápido, cubre el 99%.
- **Reconciliador en segundo plano (sweeper):** lento, cubre el 1% restante.
- **Marca de "ya hecho" (`archivedAt`):** evita que los dos caminos hagan el trabajo dos veces.

No es que el webhook sea "malo" — sigue siendo el camino principal. El sweeper existe porque en producción **siempre** va a haber algún caso raro que el camino feliz no cubre, y en healthcare no nos podemos dar el lujo de "pues si se pierde, mala suerte". El sweeper garantiza que **eventualmente todo se archiva**, sin importar qué se rompa en el camino.

---

## 5. ¿Y si rediseñamos la secuencia lógica? ¿Seguimos necesitando el sweeper?

### Propuesta de rediseño

1. LISA crea un registro de Mensaje con `requestId`, que representa un potencial mensaje enviado. En este momento ya debería existir un registro en la BD.
2. LISA le hace la petición a Spruce de enviar el mensaje con el `requestId` existente. Si Spruce responde "OK" entonces LISA actualiza el registro como enviado.
3. Spruce manda el SMS.
4. Spruce ejecuta el webhook respondiendo con el ID de la conversación. Para entonces el registro ya debe existir, porque estamos usando `async/await` en el código — entonces los pasos 2, 3 y 4 no debieron ejecutarse hasta que el guardado en la BD fue exitoso.
5. Continuamos con el resto de los pasos.

### ¿Podemos hacer esto? Sí — y de hecho ya lo hicimos

Tu propuesta es **exactamente** lo que el primer commit del ticket implementó (`a5d014cb` — *"persist requestId immediately to prevent archive race condition"*). Antes de ese commit el código guardaba el mensaje **después** de llamar a Spruce. Cambiarlo a guardar **antes** cierra la carrera principal — el ~99% de los casos.

Y tienes razón en que `async/await` garantiza el orden: si `await message.save()` se ejecuta antes de `await spruce.send()`, el registro **sí o sí** existe cuando el webhook llega.

**Pero el sweeper sigue siendo necesario.** Y la razón no es la race condition. Es otra cosa.

### Los problemas que el redesign **NO** arregla

#### 1. Spruce reentrega webhooks por su cuenta

Esto **no es una race condition nuestra** — es comportamiento de Spruce. Cuando Spruce manda un webhook y por cualquier razón no recibe un `200 OK` rápido (timeout, glitch de red, lo que sea), **vuelve a mandarlo**. A veces dos veces, a veces más.

Sin protección, LISA procesaría el mismo webhook varias veces:
- Pondría **dos** notas *"Processed by LISA"* en la conversación.
- Trataría de archivar dos veces (la segunda fallaría feo).

Por más perfecto que sea tu orden de guardado, **esto sigue pasando**. Por eso tuvimos que añadir la marca `spruce.archivedAt` como bandera de *"esto ya se hizo, no lo vuelvas a hacer"*.

#### 2. El webhook simplemente nunca llega

Esta es la grande. Hay varios escenarios donde el webhook **no llega en absoluto**:

- **Spruce tiene un fallo de entrega** — su sistema de webhooks se cae, su cola se atasca, hay un problema de red entre ellos y nosotros.
- **LISA está caída** cuando el webhook llega — Spruce intenta reentregar un par de veces y se rinde.
- **El proceso de LISA crashea a mitad** de procesar el webhook — recibimos el aviso pero morimos antes de archivar.
- **Mongo tiene un error transitorio** justo cuando vamos a archivar.

En **todos estos casos**, tu redesign perfecto no cambia nada. El mensaje está bien guardado, todo está bien ordenado, pero **nadie nos avisó nunca de que tocaba archivar**. La conversación se queda colgada para siempre en la bandeja del equipo.

#### 3. La microventana sigue existiendo (aunque más chica)

Aún con tu orden, hay un parpadeo de milisegundos entre que Spruce responde *"OK enviado"* y que LISA termina de actualizar el mensaje a "estado enviado". Si el webhook llega justo en ese parpadeo, podría ver el mensaje en un estado intermedio. Es un riesgo pequeñísimo, pero existe.

### Por qué un sweeper es la única solución de verdad

Los problemas 1 y 3 los podemos mitigar con código (la bandera `archivedAt` y guardar bien). **Pero el problema 2 es estructural:** *"qué pasa cuando un sistema externo no nos avisa de algo que esperábamos."*

Solo hay dos formas de resolver eso:

**Opción A — Confiar y rezar:** asumir que el webhook siempre llega. Cuando no llega, perdemos el mensaje y alguien tiene que limpiarlo a mano. Mala opción en healthcare.

**Opción B — Reconciliación periódica (el sweeper):** cada cierto tiempo, **nosotros** vamos y preguntamos a nuestra propia base: *"¿hay algún mensaje que se quedó sin archivar?"* Si sí, lo arreglamos. No dependemos de que nos avisen.

Esto es un patrón muy común en sistemas distribuidos y se llama **eventual consistency** con un *reconciler*. La filosofía es:

> *"Los webhooks son optimistas — funcionan casi siempre y son rápidos. La reconciliación es pesimista — asume que algo se perdió y lo busca."*

### Resumen final

| Problema | ¿El redesign lo arregla? |
|---|---|
| Race condition entre `save()` y webhook | **Sí** ✅ (es lo que hicimos en `a5d014cb`) |
| Spruce reentrega el mismo webhook | **No** — necesitamos la bandera `archivedAt` |
| El webhook nunca llega (caídas, fallos de red, Spruce-side) | **No** — necesitamos el sweeper |
| Microventana de milisegundos | **Casi** — la bandera `archivedAt` la cierra del todo |

**La regla de oro:** *cualquier sistema que dependa de un webhook externo necesita un mecanismo de reconciliación, porque tarde o temprano el webhook se pierde.* No es cuestión de **si** pasa, es cuestión de **cuándo**.

El redesign propuesto es la base correcta — y de hecho ya está en el código. El sweeper es la red de seguridad que nos protege de lo que **no controlamos** (Spruce, la red, nuestras propias caídas).
