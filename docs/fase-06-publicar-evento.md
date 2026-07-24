# Fase 6 · Publicar el evento (Redpanda)

**Rama**: `fase-6`
**Lo que vas a lograr**: conectar la app a Redpanda, definir el evento `MovimientoRegistrado`, y hacer que `transfer` lo publique en vez de llamar a `movement` directo. Después de esta fase, `transfer` deja de conocer a `movement`.

---

## Parte 1 — La dependencia y la conexión

Agrega Spring for Apache Kafka al `build.gradle.kts`:

```kotlin title="build.gradle.kts (dependencies)"
implementation("org.springframework.kafka:spring-kafka")
```

!!! note "Kafka y Redpanda hablan el mismo idioma"
    Usamos la librería de Kafka de Spring aunque el broker sea Redpanda. Redpanda implementa el protocolo de Kafka, así que del lado del código no cambia nada: la misma librería, las mismas anotaciones. Solo cambian las propiedades de conexión.

!!! abstract "¿Por qué Redpanda y no Kafka o RabbitMQ?"
    La pregunta trampa es la primera parte: **sí es Kafka**. Lo que escribes es el cliente de Kafka. Redpanda solo es otra implementación del broker, escrita en C++, sin JVM y sin Zookeeper, un binario que se levanta fácil local y tiene capa serverless en la nube. El mismo código apunta a un Apache Kafka real cambiando el `bootstrap-servers`.

    RabbitMQ sí es otro modelo. Kafka y Redpanda son un **log**: los eventos se guardan en orden, se retienen, cada grupo lleva su offset y puede releer desde el principio. RabbitMQ es una **cola**: el broker enruta y el mensaje se borra al confirmarse. Para lo nuestro, un hecho que varios consumidores independientes quieren y que un consumidor nuevo podría releer entero, el log encaja mejor. Con Rabbit tendrías que armar a mano el replay y la entrega a varios grupos. Rabbit brilla en colas de tareas y ruteo complejo; ahí sería la mejor opción.

Configura Redpanda Serverless. Los secretos van por variables de entorno, igual que Supabase:

```yaml title="src/main/resources/application.yaml (Kafka/Redpanda)"
spring:
  kafka:
    bootstrap-servers: ${REDPANDA_BOOTSTRAP}
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: SCRAM-SHA-256
      sasl.jaas.config: >-
        org.apache.kafka.common.security.scram.ScramLoginModule required
        username="${REDPANDA_USER}" password="${REDPANDA_PASSWORD}";
    # Productor: la clave como texto, el valor como JSON.
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

!!! abstract "Concepto al paso: SASL_SSL"
    Redpanda Serverless pide autenticarse. `SASL_SSL` significa dos cosas: la conexión va cifrada (SSL) y te identificas con usuario y contraseña (SASL, con mecanismo `SCRAM-SHA-256`). Esa línea larga del `jaas.config` es cómo Spring le pasa esas credenciales al cliente de Kafka. Si usas Redpanda local con Docker, esto no hace falta: basta el `bootstrap-servers`.

!!! abstract "Spring al paso: serializar el valor como JSON"
    Para viajar por el broker, el evento se convierte en bytes. Con `JsonSerializer`, Spring convierte tu objeto Kotlin a JSON automáticamente. Del otro lado, el consumidor lo reconstruye. Así publicas objetos de tu dominio sin escribir la conversión a mano.

## Parte 2 — El evento

El evento es un hecho de la feature `movement`, así que su definición vive en `movement/domain`. Es un simple `data class`, y es el **contrato compartido** entre quien lo publica (`transfer`) y quienes lo consumen.

```kotlin title="movement/domain/MovimientoRegistrado.kt"
package com.baqjug.wallet.movement.domain

import java.math.BigDecimal
import java.time.Instant
import java.util.UUID

data class MovimientoRegistrado(
    val fromId: UUID,
    val toId: UUID,
    val amount: BigDecimal,
    val occurredAt: Instant = Instant.now()
)
```

!!! note "El evento se cuenta en pasado"
    El nombre importa: `MovimientoRegistrado`, no `RegistrarMovimiento`. Es un hecho consumado, no una orden. Quien lo publica está diciendo "esto ya pasó", y no le importa quién lo escuche.

## Parte 3 — El publicador

`transfer` publica el evento a través de un pequeño componente que envuelve el `KafkaTemplate`.

```kotlin title="transfer/messaging/MovimientoPublisher.kt"
package com.baqjug.wallet.transfer.messaging

import com.baqjug.wallet.movement.domain.MovimientoRegistrado
import org.springframework.kafka.core.KafkaTemplate
import org.springframework.stereotype.Component

@Component
class MovimientoPublisher(
    private val kafkaTemplate: KafkaTemplate<String, Any>
) {
    fun publish(evento: MovimientoRegistrado) {
        // La clave es la cuenta de origen: los eventos de una misma cuenta
        // mantienen el orden entre ellos.
        kafkaTemplate.send(TOPIC, evento.fromId.toString(), evento)
    }

    companion object {
        const val TOPIC = "wallet.movements"
    }
}
```

!!! abstract "Spring al paso: `KafkaTemplate`"
    `KafkaTemplate` es la herramienta de Spring para publicar. `send(topic, clave, valor)` manda el evento al topic. Spring lo autoconfigura a partir de las propiedades que pusiste; tú solo lo inyectas y lo usas.

!!! abstract "Concepto al paso: la clave del mensaje y el orden"
    El segundo argumento, `evento.fromId.toString()`, es la **clave**. El broker garantiza el orden solo dentro de una misma partición, y los mensajes con la misma clave caen en la misma partición. Usando la cuenta de origen como clave, todos los movimientos de una cuenta se procesan en orden. Distintas cuentas pueden ir en paralelo.

!!! abstract "Kotlin al paso: `companion object`"
    Un `companion object` guarda cosas que pertenecen a la clase, no a cada instancia, como las `static` de Java. Acá metemos la constante `TOPIC` para no repetir el string mágico y tenerlo en un solo lugar.

## Parte 4 — Cortar la costura

Ahora el cambio que le da sentido a todo. ¿Te acuerdas de la rama `Success` de la Fase 4, la que llamaba directo a `movementService.record(...)`? **Esa** era la costura. Acá la cortamos: esa misma rama, en vez de llamar a `movement`, publica el evento y se desentiende.

```kotlin title="transfer/domain/TransferService.kt" hl_lines="7 8 14 22 23 24 25 26 27 28 29"
package com.baqjug.wallet.transfer.domain

import com.baqjug.wallet.account.domain.AccountNotFoundException
import com.baqjug.wallet.account.domain.AccountService
import com.baqjug.wallet.account.domain.InsufficientFundsException
import com.baqjug.wallet.account.domain.MoveResult
import com.baqjug.wallet.movement.domain.MovimientoRegistrado
import com.baqjug.wallet.transfer.messaging.MovimientoPublisher
import org.springframework.stereotype.Service

@Service
class TransferService(
    private val accountService: AccountService,
    private val publisher: MovimientoPublisher
) {

    fun transfer(request: TransferRequest) {
        require(request.fromId != request.toId) {
            "No puedes transferir a la misma cuenta"
        }
        when (val result = accountService.moveMoney(request.fromId, request.toId, request.amount)) {
            is MoveResult.Success ->
                publisher.publish(
                    MovimientoRegistrado(
                        fromId = request.fromId,
                        toId = request.toId,
                        amount = request.amount
                    )
                )
            is MoveResult.InsufficientFunds -> throw InsufficientFundsException()
            is MoveResult.AccountNotFound -> throw AccountNotFoundException(result.id)
        }
    }
}
```

!!! danger "Compara con la Fase 4"
    En la Fase 4, la rama `Success` inyectaba `MovementService` y lo llamaba (`movementService.record(...)`). Ahora esa misma rama inyecta `MovimientoPublisher` y publica el evento. `transfer` **ya no conoce** a `movement`, ni a `notification`, ni a nadie: lo único que importa de `movement` es el `data class` del evento, que es el contrato compartido. Ese cambio de dependencia —de `MovementService` a `MovimientoPublisher`— es el desacople, hecho código.

!!! note "Y el test de la Fase 4 sigue guiándote"
    Aquel test mockeaba `MovementService` y verificaba que se llamara `record`. Al cambiar la dependencia por `MovimientoPublisher`, solo ajustas el mock: `mockk<MovimientoPublisher>(relaxed = true)` en vez del de `movement`, y `verify(exactly = 1) { publisher.publish(any()) }`. La lógica que pruebas es la misma; cambió con quién habla `transfer`.

## Parte 5 — Ver el evento salir

Arranca la app y haz una transferencia como en la Fase 3. En el panel de Redpanda, en el topic `wallet.movements`, ves aparecer el evento en JSON. Nadie lo consume todavía, pero ya está publicado y guardado en el log del broker, esperando.

## Cierre de la fase

```bash
./gradlew compileKotlin
git add .
git commit -m "fase-6: transfer publica MovimientoRegistrado en Redpanda y suelta a movement"
git branch fase-6
```

El evento ya viaja. En la [Fase 7](fase-07-consumir-evento.md) lo consumimos desde el otro lado: `notification` avisa y `movement` guarda, cada uno por su cuenta, sin que `transfer` sepa que existen.
