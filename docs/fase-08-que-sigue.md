# Fase 8 · De demo a producción

**Rama**: no hay commit obligatorio, es referencia. Pero acá sí hay código, porque estos tres temas se entienden mejor viéndolos que oyéndolos.

Lo que armamos funciona y enseña bien los conceptos, pero tiene atajos que conviene nombrar en voz alta. No los escondí; los dejé para acá: outbox, idempotencia y cola de mensajes muertos. Son los tres que separan un demo de algo que aguanta plata en producción.

---

## 1. El patrón outbox: que el evento y el dato nunca se separen

Mira el orden que quedó en `transfer`: primero movemos la plata en la base (transacción), después publicamos el evento al broker. Son dos sistemas distintos, la base y Redpanda, y no comparten transacción. ¿Qué pasa si la plata se guarda pero el proceso se muere antes de publicar? La transferencia ocurrió y nadie se enteró. Al revés es peor: si publicaras antes de guardar y el guardado falla, avisaste de algo que no pasó.

Esto se llama el problema del **doble guardado** (dual write): escribir en dos lados sin una transacción que los cubra a ambos.

!!! abstract "Nivel senior: ¿y por qué no meto los dos en una sola transacción?"
    Es la primera pregunta que salta: "pongo el `moveMoney` y el `kafkaTemplate.send` dentro del mismo `@Transactional` y ya". No sirve. Una transacción de base de datos solo cubre **la base**; el broker es otro sistema, con su propia noción de "confirmado", y no comparten commit. Sí existe un mecanismo para coordinar dos sistemas —transacciones distribuidas, XA / *two-phase commit*—, pero es pesado, frágil bajo fallos y Kafka no lo soporta bien. La salida pragmática no es coordinar dos sistemas, sino **convertir el problema de dos sistemas en uno de un solo sistema** (la base). Eso es, exactamente, lo que hace el outbox.

!!! abstract "Concepto al paso: el patrón outbox"
    La idea es no publicar directo al broker. En vez de eso, escribes el evento en una tabla `outbox` **dentro de la misma transacción** que mueve la plata. O se guardan las dos cosas, o ninguna: la base sí sabe hacer eso. Después, un proceso aparte lee esa tabla y publica al broker con reintentos, marcando cada fila como enviada. El evento y el cambio de datos quedan pegados.

La tabla:

```sql title="db/migration/V3__create_outbox.sql"
CREATE TABLE outbox (
    id         UUID         NOT NULL,
    topic      VARCHAR(255) NOT NULL,
    msg_key    VARCHAR(255) NOT NULL,
    payload    TEXT         NOT NULL,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT now(),
    sent_at    TIMESTAMPTZ,
    CONSTRAINT pk_outbox PRIMARY KEY (id)
);
```

Primero, la fila `outbox` es una **entidad JPA** como cualquier otra. Va en `transfer/messaging` (es plomería de mensajería, no lógica de negocio):

```kotlin title="transfer/messaging/OutboxEvent.kt"
package com.baqjug.wallet.transfer.messaging

import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.Id
import jakarta.persistence.Table
import java.time.Instant
import java.util.UUID

@Entity
@Table(name = "outbox")
class OutboxEvent(
    @Column(nullable = false)
    val topic: String,

    @Column(name = "msg_key", nullable = false)
    val msgKey: String,

    @Column(nullable = false)
    val payload: String,

    @Column(name = "sent_at")
    var sentAt: Instant? = null,

    @Column(name = "created_at", nullable = false)
    val createdAt: Instant = Instant.now(),

    @Id
    val id: UUID = UUID.randomUUID()
)
```

Y su repositorio, con una consulta que trae solo lo que **falta por enviar**:

```kotlin title="transfer/messaging/OutboxRepository.kt"
package com.baqjug.wallet.transfer.messaging

import org.springframework.data.jpa.repository.JpaRepository
import java.util.UUID

interface OutboxRepository : JpaRepository<OutboxEvent, UUID> {
    fun findBySentAtIsNullOrderByCreatedAt(): List<OutboxEvent>
}
```

!!! abstract "Spring al paso: la consulta sale del nombre del método"
    `findBySentAtIsNullOrderByCreatedAt()` no la implementas: Spring Data la genera a partir del **nombre**. Léelo como frase: "los que tienen `sentAt` en null, ordenados por `createdAt`" — o sea, lo pendiente de publicar, del más viejo al más nuevo.

Ahora sí, mira **qué cambió** en `transfer` respecto a la Fase 6, y por qué:

- **Antes** (Fase 6), en la rama `Success`, `transfer` publicaba directo al broker: `publisher.publish(evento)`. Dos sistemas —base y broker— sin una transacción que los cubra a ambos.
- **Ahora**, en vez de publicar, **guarda una fila en la tabla `outbox`** dentro de la **misma transacción** que mueve la plata. Por eso el método es `@Transactional`: mover el saldo y escribir el outbox se confirman **juntos** (o se deshacen juntos). Así el evento no se puede perder: si la transferencia quedó guardada, su fila de outbox también.
- El evento se guarda como **texto JSON** con `mapper.writeValueAsString(evento)`, porque la columna `payload` es `TEXT`. Ese `mapper` es un `ObjectMapper` de Jackson que inyectas en el constructor. Un detalle de versiones que importa: en Spring Boot 4 (que trae Jackson 3), el `ObjectMapper` **clásico** de Jackson 2 —el que necesitan tanto este outbox como el `JsonSerializer` de spring-kafka— **no viene autoconfigurado**, así que lo declaras tú como bean. Lo dejamos junto a la configuración de Kafka, más abajo.

```kotlin title="transfer/domain/TransferService.kt (con outbox)"
@Service
class TransferService(
    private val accountService: AccountService,
    private val outbox: OutboxRepository,
    private val mapper: ObjectMapper
) {

    @Transactional
    fun transfer(request: TransferRequest) {
        require(request.fromId != request.toId) { "No puedes transferir a la misma cuenta" }
        when (val result = accountService.moveMoney(request.fromId, request.toId, request.amount)) {
            is MoveResult.Success -> {
                val evento = MovimientoRegistrado(fromId = request.fromId, toId = request.toId, amount = request.amount)
                outbox.save(
                    OutboxEvent(
                        topic = "wallet.movements",
                        msgKey = evento.fromId.toString(),
                        payload = mapper.writeValueAsString(evento)
                    )
                )
            }
            is MoveResult.InsufficientFunds -> throw InsufficientFundsException()
            is MoveResult.AccountNotFound -> throw AccountNotFoundException(result.id)
        }
    }
}
```

Y un relay que corre aparte, lee lo no enviado y publica:

```kotlin title="transfer/messaging/OutboxRelay.kt"
@Component
class OutboxRelay(
    private val outbox: OutboxRepository,
    private val kafkaTemplate: KafkaTemplate<String, String>
) {

    @Scheduled(fixedDelay = 2000)
    @Transactional
    fun publicarPendientes() {
        outbox.findBySentAtIsNullOrderByCreatedAt().forEach { fila ->
            kafkaTemplate.send(fila.topic, fila.msgKey, fila.payload)
            fila.sentAt = Instant.now()
        }
    }
}
```

!!! abstract "Spring al paso: qué hace el relay, línea por línea"
    - `@Scheduled(fixedDelay = 2000)` — Spring llama este método **solo, cada 2 segundos**, en un hilo aparte. No lo invocas tú: corre en segundo plano. (Para que funcione, la app necesita `@EnableScheduling`, que agregamos justo abajo.)
    - `@Transactional` — envuelve el barrido en una transacción, para que marcar `sent_at` se confirme bien.
    - `findBySentAtIsNullOrderByCreatedAt()` — trae las filas **pendientes** (las que aún no tienen `sent_at`), de la más vieja a la más nueva.
    - `kafkaTemplate.send(topic, key, payload)` — publica cada fila al broker. `KafkaTemplate` es la herramienta de Spring para publicar; el `payload` es el JSON que guardaste, y el `key` (la cuenta de origen) mantiene el orden por cuenta.
    - `fila.sentAt = Instant.now()` — la marca como enviada. Como el método es `@Transactional` y la fila es una entidad JPA gestionada, Hibernate detecta el cambio y hace el `UPDATE` solo (*dirty checking*); no necesitas un `save`.

Para que ese `@Scheduled` corra de verdad, la aplicación tiene que **activar el scheduler** con `@EnableScheduling`. Va en la clase principal:

```kotlin title="WalletApplication.kt"
@SpringBootApplication
@EnableScheduling
class WalletApplication

fun main(args: Array<String>) {
    runApplication<WalletApplication>(*args)
}
```

!!! warning "Sin `@EnableScheduling`, el outbox se queda mudo"
    Es un olvido silencioso, de los que cuestan encontrar. Sin esa anotación, Spring nunca arranca el scheduler, así que `publicarPendientes()` **no se ejecuta nunca**: la fila se queda en `outbox` con `sent_at` en `NULL` para siempre, nada llega a Kafka, y como el consumidor no recibe eventos, `processed_events` sale vacío. La transferencia "parece" funcionar (el saldo se mueve), pero los eventos no viajan. Si ves el `sent_at` siempre en NULL, empieza por acá.

Todo el ciclo, de la transacción al broker, se ve así:

![Diagrama de secuencia del patrón outbox: TransferService mueve el saldo y guarda una fila en la tabla outbox dentro de una sola transacción en Postgres; un OutboxRelay, cada 2 segundos, lee las filas no enviadas con FOR UPDATE SKIP LOCKED, las publica en Redpanda y marca sent_at](img/Img_5.png)

El diagrama, paso a paso:

1. **`TransferService` → Postgres (una sola transacción):** mueve el saldo **y** guarda la fila en la tabla `outbox`, las dos cosas juntas. Si algo falla, se revierten ambas.
2. **`OutboxRelay` lee (cada 2 s):** despierta por el `@Scheduled` y le pide a Postgres las filas no enviadas (`sent_at` en null). En producción, con `FOR UPDATE SKIP LOCKED` para que dos relays no tomen la misma.
3. **`OutboxRelay` → Redpanda:** publica cada fila como evento en el topic.
4. **`OutboxRelay` → Postgres:** marca la fila con `sent_at`, para no volver a publicarla.

La clave: el paso 1 es transaccional (solo la base); del 2 al 4, el relay se encarga de que lo que quedó guardado **llegue** al broker, aunque haya reintentos por el camino.

!!! note "Por qué no lo metimos en el taller en vivo"
    El outbox agrega una tabla, un relay con `@Scheduled` y serialización a mano. Para 60 minutos, habría tapado el concepto central (publicar y consumir) con plomería. Pero en algo con plata, el outbox no es opcional: es la diferencia entre "casi siempre avisa" y "siempre avisa".

!!! abstract "Nivel senior: qué garantiza el relay (y qué no)"
    El outbox te da entrega **at-least-once** (al menos una vez), no *exactly-once*. Mira el caso: el relay publica la fila al broker, pero se muere justo antes de marcar `sent_at`. Al reiniciar, la vuelve a ver como pendiente y la publica otra vez. El evento salió dos veces. Eso **no** es un defecto del outbox: es el precio de no tener transacción distribuida. Por eso el outbox y la **idempotencia** (lo que sigue) son socios: el productor promete "esto sale al menos una vez", y el consumidor promete "procesarlo dos veces no cambia el resultado". Juntos te dan el efecto de *exactly-once* sin la magia que no existe.

!!! abstract "Nivel senior: el orden y varias instancias del relay"
    Dos detalles que muerden en producción. El **orden**: publicamos por `created_at` y usamos `msg_key` (la cuenta de origen), así los eventos de una misma cuenta salen en orden y caen en la misma partición. Y las **instancias**: si corres la app replicada, dos relays podrían tomar la misma fila y publicarla doble. Se resuelve con `SELECT ... FOR UPDATE SKIP LOCKED`: cada relay agarra un lote de filas y las bloquea, y los demás **se saltan** las que ya están bloqueadas en vez de esperarlas. Con Spring Data lo expresas con `@Lock(LockModeType.PESSIMISTIC_WRITE)` y `@QueryHints` para el *skip locked*.

!!! abstract "Nivel senior: polling vs CDC (Debezium)"
    Nuestro relay hace **polling**: pregunta cada 2 segundos "¿hay filas nuevas?". Simple y suficiente para la mayoría, a costa de algo de latencia y de consultar la tabla en vano cuando no hay nada. La alternativa industrial es **CDC** (Change Data Capture) con **Debezium**: en vez de sondear, lee el **log de transacciones** de Postgres (el WAL) y publica los cambios de la tabla `outbox` al broker apenas ocurren, sin polling y con menos latencia. Es más infraestructura; para un servicio mediano, el relay con `@Scheduled` cumple de sobra. Saber que Debezium existe es lo que te deja decidir con criterio, no por moda.

## 2. Idempotencia: procesar el mismo evento dos veces sin romperlo

Un broker puede reentregar un evento. No es un bug, es cómo funciona: ante un reinicio o un rebalanceo de particiones, prefiere entregar de más que perder algo. Eso quiere decir que tu consumidor tiene que aguantar recibir el mismo evento dos veces sin duplicar el efecto. Hoy, el listener de `movement` guardaría el movimiento repetido.

!!! abstract "Concepto al paso: idempotencia"
    Una operación es idempotente si ejecutarla una vez o cinco veces deja el mismo resultado. Apagar una luz que ya está apagada no cambia nada: eso es idempotente. Queremos que "guardar este movimiento" se comporte igual.

La forma directa: darle un `id` único a cada evento y llevar registro de cuáles ya procesaste.

```kotlin title="movement/domain/MovimientoRegistrado.kt (con id)" hl_lines="2"
data class MovimientoRegistrado(
    val eventId: UUID = UUID.randomUUID(),
    val fromId: UUID,
    val toId: UUID,
    val amount: BigDecimal,
    val occurredAt: Instant = Instant.now()
)
```

!!! danger "Ojo: al agregar `eventId`, construye con nombres"
    Metimos `eventId` como **primer** campo, con valor por defecto. Si en algún lado construyes el evento por **posición** —`MovimientoRegistrado(request.fromId, ...)`—, ahora ese primer argumento se iría a `eventId` y **rompe**. La regla: construye siempre con **argumentos con nombre** —`MovimientoRegistrado(fromId = ..., toId = ..., amount = ...)`— y deja que `eventId` se autogenere. Lo mismo si mañana agregas otro campo: con nombres, nada se corre de lugar. (Por eso el `TransferService` de la Fase 6 y el outbox de arriba ya construyen con nombres.)

```sql title="db/migration/V4__create_processed_events.sql"
CREATE TABLE processed_events (
    event_id     UUID        NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT pk_processed_events PRIMARY KEY (event_id)
);
```

Necesitas dos piezas nuevas: la **entidad** que marca lo ya procesado y su **repositorio** (mapean la tabla `processed_events`, en `movement/messaging`):

```kotlin title="movement/messaging/ProcessedEvent.kt"
package com.baqjug.wallet.movement.messaging

import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.Id
import jakarta.persistence.Table
import java.time.Instant
import java.util.UUID

@Entity
@Table(name = "processed_events")
class ProcessedEvent(
    @Id
    @Column(name = "event_id")
    val eventId: UUID,

    @Column(name = "processed_at", nullable = false)
    val processedAt: Instant = Instant.now()
)
```

```kotlin title="movement/messaging/ProcessedEventRepository.kt"
package com.baqjug.wallet.movement.messaging

import org.springframework.data.jpa.repository.JpaRepository
import java.util.UUID

interface ProcessedEventRepository : JpaRepository<ProcessedEvent, UUID>
```

Y ahora sí el listener, con `movementService` y `processedRepo` inyectados, chequea, procesa y marca, todo en una transacción:

```kotlin title="movement/messaging/MovimientoPersistenceListener.kt (idempotente)"
@Component
class MovimientoPersistenceListener(
    private val movementService: MovementService,
    private val processedRepo: ProcessedEventRepository
) {

    @KafkaListener(topics = ["wallet.movements"], groupId = "movement")
    @Transactional
    fun onMovimiento(evento: MovimientoRegistrado) {
        if (processedRepo.existsById(evento.eventId)) {
            return // ya lo procesé, lo ignoro
        }
        movementService.record(evento.fromId, evento.toId, evento.amount)
        processedRepo.save(ProcessedEvent(evento.eventId))
    }
}
```

!!! note "Por qué en la misma transacción"
    Guardar el movimiento y marcar el evento como procesado tienen que confirmarse juntos. Si guardaras el movimiento pero el proceso muere antes de marcar el evento, en la reentrega lo guardarías de nuevo. La transacción los pega: o pasan los dos, o ninguno.

## 3. Cola de mensajes muertos (DLQ)

¿Qué pasa cuando un evento revienta el consumidor una y otra vez? Un JSON corrupto, un dato imposible. Si lo reintentas para siempre, ese evento tranca la partición y bloquea a los que vienen detrás.

!!! abstract "Concepto al paso: dead letter queue"
    Una DLQ (cola de mensajes muertos) es un topic aparte a donde mandas los eventos que fallaron demasiadas veces. En vez de reintentar infinito o botar el evento, lo apartas para revisarlo después, y el consumidor sigue con el resto. En Kafka la convención es un topic con sufijo `.DLT` (dead letter topic).

Spring for Kafka lo hace con un bean. Reintenta unas cuantas veces y, si sigue fallando, publica el evento al `.DLT`:

```kotlin title="config/KafkaErrorConfig.kt"
import com.fasterxml.jackson.databind.ObjectMapper
// ...demás imports

@Configuration
class KafkaErrorConfig {

    // Spring Boot 4 (Jackson 3) no autoconfigura este ObjectMapper clásico (Jackson 2);
    // lo necesitan el payload del outbox (TransferService) y el JsonSerializer de
    // spring-kafka, que aún usan Jackson 2.
    @Bean
    fun objectMapper(): ObjectMapper = ObjectMapper().findAndRegisterModules()

    @Bean
    fun errorHandler(kafkaTemplate: KafkaTemplate<Any, Any>): DefaultErrorHandler {
        // Publica a "wallet.movements.DLT" tras agotar los reintentos.
        val recoverer = DeadLetterPublishingRecoverer(kafkaTemplate)
        // 3 reintentos, esperando 1 segundo entre cada uno.
        val backOff = FixedBackOff(1000L, 3L)
        return DefaultErrorHandler(recoverer, backOff)
    }
}
```

!!! abstract "Spring al paso: por qué declaras tú el `ObjectMapper`"
    En Spring Boot 4 la autoconfiguración de Jackson pasó a **Jackson 3**. Pero el `ObjectMapper` de `com.fasterxml.jackson.databind` (Jackson **2**) —el que usan el outbox y el `JsonSerializer`/`JsonDeserializer` de spring-kafka— ya no queda registrado solo. Al declararlo como bean, Spring lo inyecta donde se pida (por ejemplo, en el constructor del `TransferService`). `findAndRegisterModules()` engancha los módulos que tengas en el classpath, entre ellos el `jackson-datatype-jsr310` de la Fase 7 —así el `Instant` del evento se serializa bien—.

!!! abstract "Spring al paso: cómo entra este bean solo"
    Spring Boot detecta que declaraste un `DefaultErrorHandler` y se lo conecta a los `@KafkaListener` sin que hagas nada más. A partir de ahí, cualquier excepción en un listener pasa por esta política: reintenta con el backoff, y al agotarse manda el evento al `.DLT`. Después conectas otro consumidor a ese `.DLT` para alertar o inspeccionar.

### Cómo verlo en una demo (sin Slack)

Para mostrar la DLQ en vivo tienes dos caminos, y ninguno necesita Slack ni infraestructura pesada:

- **El más simple, en la consola**: un `@KafkaListener` extra suscrito al `.DLT` que solo **loguea** lo que llega. Mandas un evento "envenenado" (un JSON corrupto o un dato imposible que haga fallar al consumidor de `movement`), se agotan los reintentos, y ves el mensaje muerto aparecer en la consola.
- **El más visual, sin código extra**: como ya usas **Redpanda**, ábrete su panel — al lado de `wallet.movements` va a aparecer el topic **`wallet.movements.DLT`** llenándose con los que fallaron. Señalas ahí y se explica solo. (Si corres el broker local con Docker, una UI como **Redpanda Console**, **AKHQ** o **Kafdrop** hace lo mismo en el navegador.)

El listener de demo:

```kotlin title="notification/messaging/DeadLetterListener.kt (solo para la demo)"
package com.baqjug.wallet.notification.messaging

import org.apache.kafka.clients.consumer.ConsumerRecord
import org.slf4j.LoggerFactory
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.stereotype.Component

@Component
class DeadLetterListener {
    private val log = LoggerFactory.getLogger(javaClass)

    @KafkaListener(topics = ["wallet.movements.DLT"], groupId = "dlt-demo")
    fun onDead(record: ConsumerRecord<String, String>) {
        log.warn("💀 Mensaje muerto en el DLT: key={} value={}", record.key(), record.value())
    }
}
```

El guion de la demo: mandas el evento envenenado, el consumidor de `movement` falla 3 veces (el `FixedBackOff`), y el evento aparece en `wallet.movements.DLT` —en tu consola por el `DeadLetterListener`, o en el panel de Redpanda—. Eso es justo lo que quieres mostrar: un mensaje malo no tranca la cola, se aparta y los demás siguen.

---

## Probar los escenarios

Para ejercitar todo esto sin adivinar, deja un archivo `test.http` en el proyecto: IntelliJ lo corre con el ▶️ al lado de cada request, y también lo puedes importar a Postman.

```http title="test.http"
### 1. Consultar el saldo de Elena
GET http://localhost:8080/api/accounts/11111111-1111-1111-1111-111111111111

### 2. Transferir (dispara el evento: outbox -> relay -> consumidores)
POST http://localhost:8080/api/transfers
Content-Type: application/json

{
  "fromId": "11111111-1111-1111-1111-111111111111",
  "toId": "22222222-2222-2222-2222-222222222222",
  "amount": 15000.00
}

### 3. Consultar el saldo de Geovanny (debió subir)
GET http://localhost:8080/api/accounts/22222222-2222-2222-2222-222222222222
```

### Verificar en la base de datos

Abre el **SQL Editor** de Supabase (o cualquier cliente Postgres) y corre estas consultas para comprobar, con tus ojos, que todo pasó de verdad:

```sql
-- Saldos actuales: Elena debió bajar, Geovanny subir.
SELECT id, owner, balance FROM accounts ORDER BY owner;

-- El movimiento quedó registrado (uno por transferencia).
SELECT from_id, to_id, amount, occurred_at
FROM movements ORDER BY occurred_at DESC LIMIT 10;

-- Outbox: la fila del evento. Al principio con sent_at en NULL; un par
-- de segundos después el relay la marca (ya la publicó al broker).
SELECT id, topic, msg_key, sent_at, created_at
FROM outbox ORDER BY created_at DESC LIMIT 10;

-- ¿Queda algo sin publicar? (debería bajar a 0 rápido)
SELECT count(*) AS pendientes FROM outbox WHERE sent_at IS NULL;

-- Idempotencia: los eventos que el consumidor ya procesó.
SELECT event_id, processed_at FROM processed_events ORDER BY processed_at DESC LIMIT 10;
```

- **Outbox** — tras el `POST`, la fila aparece en `outbox` y su `sent_at` se llena en segundos; en `movements` ves el registro y en Mailpit el correo.
- **Idempotencia** — para forzar una reentrega, **reinicia la app** (con `auto-offset-reset: earliest` el grupo vuelve a leer los eventos). El evento llega otra vez, pero **no** se duplica. Compruébalo:

```sql
-- Si sale alguna fila, hubo duplicado. No debería salir ninguna.
SELECT from_id, to_id, amount, count(*)
FROM movements
GROUP BY from_id, to_id, amount
HAVING count(*) > 1;
```

El `eventId` ya estaba en `processed_events`, así que el listener lo ignoró y no guardó de nuevo.

### Probar la DLQ

La DLQ solo se llena cuando el consumidor **falla** con un mensaje una y otra vez. Como el endpoint valida los datos (no te deja mandar un evento malo), el mensaje "envenenado" hay que meterlo **directo al topic**. Dos caminos:

**Opción A — que el listener falle a propósito (la más confiable para una demo).** En el listener de `movement`, agrega temporalmente un centinela:

```kotlin
if (evento.amount < BigDecimal.ZERO) throw IllegalStateException("evento envenenado para la demo")
```

y produce un evento con monto negativo directo al topic (por eso `rpk`, no el endpoint):

```bash
echo '{"eventId":"00000000-0000-0000-0000-000000000000","fromId":"11111111-1111-1111-1111-111111111111","toId":"22222222-2222-2222-2222-222222222222","amount":-1,"occurredAt":"2026-01-01T00:00:00Z"}'   | rpk topic produce wallet.movements --key 11111111-1111-1111-1111-111111111111
```

El listener lanza, el `DefaultErrorHandler` reintenta 3 veces (el `FixedBackOff`) y, agotados los intentos, el `DeadLetterPublishingRecoverer` publica el evento en **`wallet.movements.DLT`**.

**Opción B — JSON corrupto.** Si mandas un JSON malformado, el error es de **deserialización** (ocurre antes de entrar al listener). Para que ese tipo de error llegue al `.DLT` en vez de quedar en bucle, el value-deserializer debe ir envuelto en un `ErrorHandlingDeserializer` (un paso extra de configuración). Por eso, para el taller, la Opción A es más directa.

**Cómo verlo llegar al `.DLT`:**

```bash
# Consumir el topic de mensajes muertos
rpk topic consume wallet.movements.DLT
```

También aparece en el **panel de Redpanda** (el topic `wallet.movements.DLT` con el evento dentro), o en el log si tienes el `DeadLetterListener` de arriba. Eso confirma lo que buscabas: un mensaje malo no tranca la cola —se aparta— y los demás siguen su curso.

## Lo que quedó por fuera, en una línea cada uno

- **Concurrencia sobre la misma cuenta.** Dos transferencias en paralelo sobre la misma cuenta pueden pisarse. Se resuelve con bloqueo optimista: una columna `@Version` en `Account` hace que la segunda transacción en confirmar falle y se reintente con el saldo fresco.
- **Separar los servicios de verdad.** Productor y consumidores viven hoy en la misma app por comodidad. Como `transfer` solo publica un evento y no conoce a nadie, sacar `notification` a su propio despliegue es mover el listener a otro proyecto suscrito al mismo topic. El evento en `movement/domain` es el contrato compartido.
- **Observabilidad.** Cuando un evento cruza servicios, querrás un `trace-id` que viaje con él, para seguir una transferencia desde el `POST` hasta la notificación aunque hayan pasado por tres procesos.

## Lo que te llevas

Construiste una wallet que mueve plata de forma transaccional, la expusiste por REST, y desacoplaste el registro y la notificación con eventos sobre Redpanda, con consumidores independientes que reaccionan sin conocerse. Y ahora sabes qué le falta para producción, con código, no con handwaving: outbox para no perder eventos, idempotencia para no duplicarlos, y DLQ para no atascarte con uno malo.

¿Quieres exprimir más el lado Kotlin? En la [Fase 9](fase-09-coroutines.md) —avanzada y opcional— tomamos el consumidor bloqueante de la Fase 7 y lo llevamos a **coroutines**, para cuando un consumidor tiene que llamar a varios servicios externos sin bloquear un hilo por cada espera.

Revisa las [Referencias](referencias.md) para seguir por tu cuenta.
