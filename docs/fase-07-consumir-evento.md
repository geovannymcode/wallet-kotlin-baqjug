# Fase 7 · Consumir el evento

**Rama**: `fase-7` = `main`
**Lo que vas a lograr**: consumir el evento `MovimientoRegistrado` desde dos lados a la vez. `notification` avisa, `movement` guarda, cada uno en su propio grupo, sin que `transfer` sepa que existen. Acá se ve, en vivo, por qué los eventos valen la pena.

---

## Parte 1 — Configurar el consumidor (5 min)

Súmale al `application.properties` el lado consumidor. El productor mandaba JSON; el consumidor lo reconstruye.

```properties title="src/main/resources/application.properties (consumer)"
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=com.baqjug.wallet.*
spring.kafka.consumer.auto-offset-reset=earliest
```

!!! abstract "Concepto al paso: `trusted.packages` y por qué existe"
    Deserializar JSON a un objeto Kotlin significa crear una clase a partir de datos que llegaron de afuera. Eso, sin control, es un riesgo de seguridad. `trusted.packages` es una lista blanca: solo se permite reconstruir clases de esos paquetes. Le decimos que confíe en las nuestras (`com.baqjug.wallet.*`).

!!! abstract "Concepto al paso: `auto-offset-reset=earliest`"
    Cuando un grupo llega por primera vez a un topic y no tiene una marca previa (offset), ¿desde dónde lee? Con `earliest`, desde el principio del log. Con `latest`, solo lo nuevo. En el taller usamos `earliest` para que un consumidor que enciendes después alcance a ver los eventos que ya publicaste.

## Parte 2 — El consumidor que notifica (10 min)

Primera feature nueva: `notification`. Solo escucha y "avisa". Acá el aviso es un log, pero en la vida real sería un correo o un push.

```kotlin title="notification/internal/MovimientoNotificationListener.kt"
package com.baqjug.wallet.notification.internal

import com.baqjug.wallet.movement.api.MovimientoRegistrado
import org.slf4j.LoggerFactory
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.stereotype.Component

@Component
class MovimientoNotificationListener {

    private val log = LoggerFactory.getLogger(javaClass)

    @KafkaListener(topics = ["wallet.movements"], groupId = "notification")
    fun onMovimiento(evento: MovimientoRegistrado) {
        log.info(
            "🔔 Notificación: se movieron {} de la cuenta {} a la {}",
            evento.amount, evento.fromId, evento.toId
        )
    }
}
```

!!! abstract "Spring al paso: `@KafkaListener`"
    `@KafkaListener` convierte un método en consumidor. `topics` dice qué escuchar, `groupId` a qué grupo pertenece. Spring se suscribe al arrancar, y cada vez que llega un evento, llama tu método con el objeto ya deserializado. Tú solo escribes qué hacer con él.

## Parte 3 — El consumidor que guarda (10 min)

Acá está lo interesante. En la Fase 4, `movement` guardaba porque `transfer` lo llamaba. Ahora `movement` guarda porque **escucha el evento**, en su propio grupo. Reusa el `MovementService` que ya tenías.

```kotlin title="movement/internal/MovimientoPersistenceListener.kt"
package com.baqjug.wallet.movement.internal

import com.baqjug.wallet.movement.api.MovementService
import com.baqjug.wallet.movement.api.MovimientoRegistrado
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.stereotype.Component

@Component
class MovimientoPersistenceListener(
    private val movementService: MovementService
) {

    @KafkaListener(topics = ["wallet.movements"], groupId = "movement")
    fun onMovimiento(evento: MovimientoRegistrado) {
        movementService.record(evento.fromId, evento.toId, evento.amount)
    }
}
```

!!! danger "Grupos distintos: los dos reciben cada evento"
    Fíjate: `notification` está en el grupo `notification` y `movement` en el grupo `movement`. Como son grupos distintos, **cada uno recibe una copia** de cada evento. Una sola transferencia dispara un aviso y un registro, en paralelo, sin que se pisen. Si ambos estuvieran en el mismo grupo, el broker le daría el evento a uno solo. Esa diferencia es el corazón del pub/sub.

!!! note "Un consumidor nuevo no toca a `transfer`"
    Este es el pago de todo el trabajo. Agregaste dos consumidores y no tocaste una línea de `transfer`. Mañana quieres antifraude: creas otro `@KafkaListener` con su grupo y listo. `transfer` ni se entera. Compara eso con el `if` de la Fase 4, donde cada cosa nueva era editar y redesplegar `transfer`.

## Parte 4 — Verlo funcionar (10 min)

Arranca la app y haz una transferencia:

```bash
curl -X POST http://localhost:8080/api/transfers \
  -H "Content-Type: application/json" \
  -d '{
        "fromId": "11111111-1111-1111-1111-111111111111",
        "toId": "22222222-2222-2222-2222-222222222222",
        "amount": 15000.00
      }'
```

En los logs ves el aviso del listener de `notification`. En Supabase, la tabla `movements` tiene una fila nueva, guardada por el listener de `movement`. Un evento, dos consumidores, cero acople con `transfer`.

!!! warning "Ojo con procesar el mismo evento dos veces"
    Si el broker reentrega un evento (pasa: reinicios, rebalanceos), el listener de `movement` podría guardar el movimiento dos veces. Hoy no lo evitamos. Cómo hacer que consumir el mismo evento varias veces no rompa nada, eso es idempotencia, y lo vemos en la [Fase 8](fase-08-que-sigue.md).

## Cierre de la fase

```bash
./gradlew test
git add .
git commit -m "fase-7: notification y movement consumen el evento en grupos separados"
git branch fase-7
git checkout -b main
```

Esta es la versión completa. Tienes una wallet que transfiere validando saldo, publica un hecho cuando algo pasa, y tiene consumidores independientes que reaccionan sin conocerse entre sí.

En la [Fase 8](fase-08-que-sigue.md) cierro con lo que falta para que esto aguante producción: idempotencia, el patrón outbox, colas de mensajes muertos, y separar de verdad los servicios.
