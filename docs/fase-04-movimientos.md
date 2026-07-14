# Fase 4 · Registro de movimientos

**Rama**: `fase-4`
**Lo que vas a lograr**: dejar registrado cada movimiento en su propia tabla, con una llamada directa desde `transfer`. Y de paso, ver por qué esa llamada directa nos va a estorbar más adelante.

Esta es la última fase de IDITEK. Al terminarla, tienes una wallet completa por REST. Es también el punto donde CaribeConf empieza a jugar.

---

## Parte 1 — El movimiento como entidad (10 min)

La feature `movement` guarda un registro por cada transferencia: de quién, a quién, cuánto y cuándo.

```kotlin title="movement/internal/Movement.kt"
package com.baqjug.wallet.movement.internal

import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.Id
import jakarta.persistence.Table
import java.math.BigDecimal
import java.time.Instant
import java.util.UUID

@Entity
@Table(name = "movements")
class Movement(
    @Column(nullable = false)
    val fromId: UUID,

    @Column(nullable = false)
    val toId: UUID,

    @Column(nullable = false, precision = 19, scale = 2)
    val amount: BigDecimal,

    @Column(nullable = false)
    val occurredAt: Instant = Instant.now(),

    @Id
    val id: UUID = UUID.randomUUID()
)
```

```kotlin title="movement/internal/MovementRepository.kt"
package com.baqjug.wallet.movement.internal

import org.springframework.data.jpa.repository.JpaRepository
import java.util.UUID

interface MovementRepository : JpaRepository<Movement, UUID>
```

La migración para su tabla:

```sql title="db/migration/V2__create_movements.sql"
CREATE TABLE movements (
    id          UUID           NOT NULL,
    from_id     UUID           NOT NULL,
    to_id       UUID           NOT NULL,
    amount      NUMERIC(19, 2) NOT NULL,
    occurred_at TIMESTAMPTZ    NOT NULL,
    CONSTRAINT pk_movements PRIMARY KEY (id)
);
```

!!! note "El nombre de las columnas"
    En la entidad la propiedad es `fromId` (camelCase) y en la tabla la columna es `from_id` (snake_case). Spring Boot mapea entre esos dos estilos por defecto. Si algún día no coincide, lo fijas con `@Column(name = "...")`.

## Parte 2 — El servicio de movimientos (5 min)

Vitrina en `api`, bodega en `internal`, como siempre.

```kotlin title="movement/api/MovementService.kt"
package com.baqjug.wallet.movement.api

import java.math.BigDecimal
import java.util.UUID

interface MovementService {
    fun record(fromId: UUID, toId: UUID, amount: BigDecimal)
}
```

```kotlin title="movement/internal/DefaultMovementService.kt"
package com.baqjug.wallet.movement.internal

import com.baqjug.wallet.movement.api.MovementService
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.math.BigDecimal
import java.util.UUID

@Service
@Transactional
class DefaultMovementService(
    private val repository: MovementRepository
) : MovementService {

    override fun record(fromId: UUID, toId: UUID, amount: BigDecimal) {
        repository.save(Movement(fromId = fromId, toId = toId, amount = amount))
    }
}
```

!!! abstract "Kotlin al paso: argumentos con nombre"
    Al construir el `Movement` escribí `fromId = fromId, toId = toId, ...`. Eso son **argumentos con nombre**: en vez de depender del orden, nombras cada parámetro. En un constructor con varios campos del mismo tipo (acá hay tres `UUID`), esto evita el clásico error de invertir dos valores sin darte cuenta.

## Parte 3 — La llamada directa desde `transfer` (10 min)

Ahora `transfer`, cuando el movimiento de plata sale bien, llama a `movement` para registrarlo:

```kotlin title="transfer/internal/DefaultTransferService.kt" hl_lines="4 8 15 16"
@Service
class DefaultTransferService(
    private val accountService: AccountService,
    private val movementService: MovementService
) : TransferService {

    override fun transfer(request: TransferRequest): MoveResult {
        require(request.fromId != request.toId) {
            "No puedes transferir a la misma cuenta"
        }
        val result = accountService.moveMoney(request.fromId, request.toId, request.amount)

        if (result is MoveResult.Success) {
            // Llamada directa y síncrona. Anota esto: acá está la costura.
            movementService.record(request.fromId, request.toId, request.amount)
        }
        return result
    }
}
```

!!! danger "Acá está la costura que abre CaribeConf"
    Fíjate qué acabamos de hacer: `transfer` ahora **conoce** a `movement` y lo llama en línea. Funciona. Pero si mañana además hay que mandar un correo, disparar un push y avisar a antifraude, todos esos van a colgarse de este mismo `if`. Y si `movement` se pone lento o falla, la transferencia se cuelga o se cae con él. Esa es exactamente la costura que en la Fase 6 vamos a cortar con eventos.

## Parte 4 — Un test que corre sin base (10 min)

Antes de cerrar, un test de la orquestación de `transfer`. Lo bueno: no necesita base ni Spring, porque le pasamos dobles de prueba hechos a mano.

```kotlin title="src/test/kotlin/.../transfer/DefaultTransferServiceTest.kt"
package com.baqjug.wallet.transfer

import com.baqjug.wallet.account.api.AccountResponse
import com.baqjug.wallet.account.api.AccountService
import com.baqjug.wallet.account.api.MoveResult
import com.baqjug.wallet.movement.api.MovementService
import com.baqjug.wallet.transfer.api.TransferRequest
import com.baqjug.wallet.transfer.internal.DefaultTransferService
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test
import java.math.BigDecimal
import java.util.UUID

class DefaultTransferServiceTest {

    // Doble de prueba: siempre dice que el movimiento salió bien.
    private val accountOk = object : AccountService {
        override fun getById(id: UUID) = AccountResponse(id, "test", BigDecimal.TEN)
        override fun moveMoney(fromId: UUID, toId: UUID, amount: BigDecimal) = MoveResult.Success
    }

    // Doble de prueba: cuenta cuántas veces le pidieron registrar.
    private class RecordingMovements : MovementService {
        var calls = 0
        override fun record(fromId: UUID, toId: UUID, amount: BigDecimal) { calls++ }
    }

    @Test
    fun `registra el movimiento cuando la transferencia se completa`() {
        val movements = RecordingMovements()
        val service = DefaultTransferService(accountOk, movements)

        val result = service.transfer(
            TransferRequest(UUID.randomUUID(), UUID.randomUUID(), BigDecimal("5.00"))
        )

        assertTrue(result is MoveResult.Success)
        assertEquals(1, movements.calls)
    }
}
```

!!! abstract "Concepto al paso: doble de prueba (test double)"
    Un doble de prueba es una implementación falsa de una dependencia, hecha para el test. Acá `accountOk` finge que el movimiento siempre sale bien, y `RecordingMovements` no guarda nada, solo cuenta las llamadas. Así pruebas la lógica de `transfer` sola, sin base y sin Spring: rápido y sin infraestructura.

!!! abstract "Kotlin al paso: nombres de test entre backticks"
    En Kotlin puedes nombrar una función con espacios si la envuelves en backticks: `` `registra el movimiento cuando...` ``. Para tests es oro: el nombre se lee como una frase y el reporte queda claro.

!!! warning "Paquetes de test movidos en Spring Boot 4"
    Si más adelante escribes tests de integración con anotaciones como `@DataJpaTest`, ojo: en Boot 4 varias se mudaron de paquete. `@DataJpaTest` ahora viene de `org.springframework.boot.data.jpa.test.autoconfigure`. Si copias un tutorial de Boot 3, los imports no compilan.

## Cierre de la fase

```bash
./gradlew test
git add .
git commit -m "fase-4: registro de movimientos por llamada directa y test de orquestación"
git branch fase-4
```

Hasta acá llega IDITEK: una wallet que consulta saldo, transfiere validando, y registra movimientos. Todo por REST, contra Supabase.

De la [Fase 5](fase-05-por-que-eventos.md) en adelante entra CaribeConf. Vamos a tomar esa llamada directa a `movement` y convertirla en un evento.
