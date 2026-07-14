# Fase 6 · Publicar el evento (Redpanda)

**Rama**: `fase-6`
**Lo que vas a lograr**: conectar la app a Redpanda, definir el evento `MovimientoRegistrado`, y hacer que `transfer` lo publique en vez de llamar a `movement` directo. Después de esta fase, `transfer` deja de conocer a `movement`.

---

## Parte 1 — La dependencia y la conexión (10 min)

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

```properties title="src/main/resources/application.properties (Kafka/Redpanda)"
spring.kafka.bootstrap-servers=${REDPANDA_BOOTSTRAP}
spring.kafka.properties.security.protocol=SASL_SSL
spring.kafka.properties.sasl.mechanism=SCRAM-SHA-256
spring.kafka.properties.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="${REDPANDA_USER}" password="${REDPANDA_PASSWORD}";

# Productor: la clave como texto, el valor como JSON.
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
```

!!! abstract "Concepto al paso: SASL_SSL"
    Redpanda Serverless pide autenticarse. `SASL_SSL` significa dos cosas: la conexión va cifrada (SSL) y te identificas con usuario y contraseña (SASL, con mecanismo `SCRAM-SHA-256`). Esa línea larga del `jaas.config` es cómo Spring le pasa esas credenciales al cliente de Kafka. Si usas Redpanda local con Docker, esto no hace falta: basta el `bootstrap-servers`.

!!! abstract "Spring al paso: serializar el valor como JSON"
    Para viajar por el broker, el evento se convierte en bytes. Con `JsonSerializer`, Spring convierte tu objeto Kotlin a JSON automáticamente. Del otro lado, el consumidor lo reconstruye. Así publicas objetos de tu dominio sin escribir la conversión a mano.

## Parte 2 — El evento (5 min)

El evento es un hecho de la feature `movement`, así que su definición vive en la vitrina, `movement/api`. Es un simple `data class`.

```kotlin title="movement/api/MovimientoRegistrado.kt"
package com.baqjug.wallet.movement.api

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

## Parte 3 — El publicador (10 min)

`transfer` publica el evento a través de un pequeño componente que envuelve el `KafkaTemplate`.

```kotlin title="transfer/internal/MovimientoPublisher.kt"
package com.baqjug.wallet.transfer.internal

import com.baqjug.wallet.movement.api.MovimientoRegistrado
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

## Parte 4 — Cortar la costura (10 min)

Ahora el cambio que le da sentido a todo. `transfer` deja de llamar a `movement` y publica el evento:

```kotlin title="transfer/internal/DefaultTransferService.kt" hl_lines="3 14 15 16 17 18 19 20"
@Service
class DefaultTransferService(
    private val accountService: AccountService,
    private val publisher: MovimientoPublisher
) : TransferService {

    override fun transfer(request: TransferRequest): MoveResult {
        require(request.fromId != request.toId) {
            "No puedes transferir a la misma cuenta"
        }
        val result = accountService.moveMoney(request.fromId, request.toId, request.amount)

        if (result is MoveResult.Success) {
            publisher.publish(
                MovimientoRegistrado(
                    fromId = request.fromId,
                    toId = request.toId,
                    amount = request.amount
                )
            )
        }
        return result
    }
}
```

!!! danger "Compara con la Fase 4"
    Antes: `transfer` importaba `MovementService` y lo llamaba. Ahora: importa solo el evento y lo publica. `transfer` **ya no conoce** a `movement`, ni a `notification`, ni a nadie. Publicó un hecho y se desentendió. Ese import que desapareció es el desacople, hecho código.

!!! note "Y el test de la Fase 4 sigue guiándote"
    Aquel test usaba un doble de `MovementService`. Al cambiar la dependencia por `MovimientoPublisher`, el test te marca dónde ajustar: le pasas un doble del publicador que cuente las publicaciones. La lógica que pruebas es la misma; cambió con quién habla `transfer`.

## Parte 5 — Ver el evento salir (5 min)

Arranca la app y haz una transferencia como en la Fase 3. En el panel de Redpanda, en el topic `wallet.movements`, ves aparecer el evento en JSON. Nadie lo consume todavía, pero ya está publicado y guardado en el log del broker, esperando.

## Cierre de la fase

```bash
./gradlew compileKotlin
git add .
git commit -m "fase-6: transfer publica MovimientoRegistrado en Redpanda y suelta a movement"
git branch fase-6
```

El evento ya viaja. En la [Fase 7](fase-07-consumir-evento.md) lo consumimos desde el otro lado: `notification` avisa y `movement` guarda, cada uno por su cuenta, sin que `transfer` sepa que existen.
