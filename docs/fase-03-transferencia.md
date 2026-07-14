# Fase 3 · Transferencias con validación

**Rama**: `fase-3`
**Lo que vas a lograr**: mover plata de una cuenta a otra con un `POST`, validando el saldo, todo dentro de una transacción que no puede dejar cuentas a medias.

---

## Parte 1 — El resultado como `sealed class` (10 min)

Una transferencia puede terminar de varias formas: se completa, no hay saldo, o una de las cuentas no existe. En vez de andar con excepciones o con un booleano pobre, modelamos el resultado con un tipo que enumera exactamente los finales posibles. Va en la `api` de `account`, porque mover plata entre cuentas es, en el fondo, una operación sobre cuentas.

```kotlin title="account/api/MoveResult.kt"
package com.baqjug.wallet.account.api

import java.util.UUID

sealed class MoveResult {
    data object Success : MoveResult()
    data object InsufficientFunds : MoveResult()
    data class AccountNotFound(val id: UUID) : MoveResult()
}
```

!!! abstract "Kotlin al paso: `sealed class`"
    Una `sealed class` es una jerarquía cerrada: sus únicas subclases son las que declaras, acá mismo. El compilador conoce toda la lista. ¿Por qué importa? Porque cuando decidas qué hacer según el resultado, si te falta un caso, Kotlin te lo marca. Es un `enum` con esteroides: cada caso puede llevar datos propios (fíjate que `AccountNotFound` carga el `id` que falló).

!!! abstract "Kotlin al paso: `data object` vs `data class`"
    `Success` e `InsufficientFunds` no cargan datos, por eso son `data object`: existe una sola instancia, como un singleton. `AccountNotFound` sí lleva un `id`, por eso es `data class`. Usa `object` cuando el caso es único y sin datos, `class` cuando necesita cargar algo.

## Parte 2 — Mover la plata, en la bodega de `account` (15 min)

Sumamos una operación a `AccountService`:

```kotlin title="account/api/AccountService.kt" hl_lines="7"
package com.baqjug.wallet.account.api

import java.math.BigDecimal
import java.util.UUID

interface AccountService {
    fun getById(id: UUID): AccountResponse
    fun moveMoney(fromId: UUID, toId: UUID, amount: BigDecimal): MoveResult
}
```

Y la implementamos. Acá está el corazón de la wallet:

```kotlin title="account/internal/DefaultAccountService.kt" hl_lines="10 11 12 13 14 15 16 17 18 19"
override fun moveMoney(fromId: UUID, toId: UUID, amount: BigDecimal): MoveResult {
    val from = repository.findById(fromId).orElse(null)
        ?: return MoveResult.AccountNotFound(fromId)
    val to = repository.findById(toId).orElse(null)
        ?: return MoveResult.AccountNotFound(toId)

    if (from.balance < amount) {
        return MoveResult.InsufficientFunds
    }

    from.balance = from.balance.subtract(amount)
    to.balance = to.balance.add(amount)
    repository.save(from)
    repository.save(to)

    return MoveResult.Success
}
```

Y este método sí escribe, así que su anotación cambia:

```kotlin title="account/internal/DefaultAccountService.kt (la clase)"
@Service
@Transactional
class DefaultAccountService(
    private val repository: AccountRepository
) : AccountService {
    // getById(...) sigue igual (puede quedar con @Transactional(readOnly = true) a nivel de método)
    // moveMoney(...) arriba
}
```

!!! abstract "Spring al paso: `@Transactional` que escribe, y por qué es clave acá"
    `@Transactional` envuelve el método en una transacción. Todo lo que pase adentro se confirma junto (commit) o se deshace junto (rollback). Si al guardar la segunda cuenta algo explota, la primera **no** queda debitada: la transacción entera se revierte. Esto es lo que garantiza que nunca quede una cuenta con plata de menos y la otra sin la de más.

!!! abstract "Kotlin al paso: el operador Elvis `?:`"
    `repository.findById(id).orElse(null) ?: return ...` se lee así: intenta obtener la cuenta; si es `null`, ejecuta lo de la derecha (retornar el resultado de "no encontrada"). El `?:` se llama operador Elvis y es el atajo de Kotlin para "si esto es nulo, entonces aquello". Corta el flujo temprano y deja el resto del método con la cuenta ya garantizada no nula.

!!! note "Sobre concurrencia (para no engañarte)"
    Este `moveMoney` es correcto para el taller, pero dos transferencias en paralelo sobre la misma cuenta podrían pisarse. En producción se resuelve con bloqueo optimista (una columna `@Version`) o pesimista. Lo menciono para que sepas que existe; implementarlo queda para la [Fase 8](fase-08-que-sigue.md).

## Parte 3 — La petición y su validación (10 min)

La feature `transfer` recibe la petición. Su DTO valida la forma de los datos antes de que lleguen a la lógica.

```kotlin title="transfer/api/TransferRequest.kt"
package com.baqjug.wallet.transfer.api

import jakarta.validation.constraints.DecimalMin
import jakarta.validation.constraints.NotNull
import java.math.BigDecimal
import java.util.UUID

data class TransferRequest(
    @field:NotNull
    val fromId: UUID,

    @field:NotNull
    val toId: UUID,

    @field:NotNull
    @field:DecimalMin(value = "0.01", message = "El monto debe ser mayor a cero")
    val amount: BigDecimal
)
```

!!! danger "El prefijo `@field:` es obligatorio en Kotlin"
    En Kotlin, una anotación de validación sobre una propiedad tiene que ir con `@field:`. Sin ese prefijo, la validación **no corre**, y lo peor es que no falla: simplemente se ignora en silencio. Es el error más común al validar DTOs en Kotlin. Si tu validación "no funciona", revisa esto primero.

## Parte 4 — El servicio y el controlador de `transfer` (10 min)

```kotlin title="transfer/api/TransferService.kt"
package com.baqjug.wallet.transfer.api

import com.baqjug.wallet.account.api.MoveResult

interface TransferService {
    fun transfer(request: TransferRequest): MoveResult
}
```

```kotlin title="transfer/internal/DefaultTransferService.kt"
package com.baqjug.wallet.transfer.internal

import com.baqjug.wallet.account.api.AccountService
import com.baqjug.wallet.account.api.MoveResult
import com.baqjug.wallet.transfer.api.TransferRequest
import com.baqjug.wallet.transfer.api.TransferService
import org.springframework.stereotype.Service

@Service
class DefaultTransferService(
    private val accountService: AccountService
) : TransferService {

    override fun transfer(request: TransferRequest): MoveResult {
        require(request.fromId != request.toId) {
            "No puedes transferir a la misma cuenta"
        }
        return accountService.moveMoney(request.fromId, request.toId, request.amount)
    }
}
```

!!! note "Fíjate en el desacople"
    `transfer` no toca la base ni sabe cómo se guardan las cuentas. Pide `AccountService` (la vitrina de `account`) y delega el movimiento. Su trabajo es orquestar el caso de uso, no manejar plata a mano.

El controlador traduce el resultado a HTTP:

```kotlin title="transfer/internal/TransferController.kt"
package com.baqjug.wallet.transfer.internal

import com.baqjug.wallet.account.api.MoveResult
import com.baqjug.wallet.transfer.api.TransferRequest
import com.baqjug.wallet.transfer.api.TransferService
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/transfers")
class TransferController(
    private val transferService: TransferService
) {

    @PostMapping
    fun transfer(@Valid @RequestBody request: TransferRequest): ResponseEntity<Any> =
        when (val result = transferService.transfer(request)) {
            is MoveResult.Success ->
                ResponseEntity.ok(mapOf("status" to "COMPLETED"))
            is MoveResult.InsufficientFunds ->
                ResponseEntity.unprocessableEntity().body(mapOf("error" to "Saldo insuficiente"))
            is MoveResult.AccountNotFound ->
                ResponseEntity.status(HttpStatus.NOT_FOUND)
                    .body(mapOf("error" to "No existe la cuenta ${result.id}"))
        }
}
```

!!! abstract "Spring al paso: `@PostMapping`, `@RequestBody`, `@Valid`"
    `@PostMapping` atiende un `POST`. `@RequestBody` toma el JSON del cuerpo y lo convierte en tu `TransferRequest`. `@Valid` dispara las validaciones que pusimos con `@field:`; si alguna falla, Spring responde `400` antes de que tu código corra.

!!! abstract "Kotlin al paso: `when` sobre una `sealed class`"
    `when` es como un `switch`, pero de verdad. Sobre una `sealed class`, Kotlin sabe cuáles son todos los casos: si te olvidas de manejar uno, **no compila**. Por eso no necesitas un `else`. Fíjate el `when (val result = ...)`: guarda el resultado en `result` y ramifica según su tipo con `is`. Cuando entra por `AccountNotFound`, ya puedes leer `result.id`.

## Parte 5 — Probar la transferencia (5 min)

```bash
curl -X POST http://localhost:8080/api/transfers \
  -H "Content-Type: application/json" \
  -d '{
        "fromId": "11111111-1111-1111-1111-111111111111",
        "toId": "22222222-2222-2222-2222-222222222222",
        "amount": 30000.00
      }'
```

Consulta los saldos después: Ana quedó en 70000 y Beto en 30000. Intenta ahora transferir 999999 y te da `422 Saldo insuficiente`, sin mover un peso.

## Cierre de la fase

```bash
./gradlew compileKotlin
git add .
git commit -m "fase-3: transferencia transaccional con validación de saldo y resultado sealed"
git branch fase-3
```

La wallet ya mueve plata de verdad. En la [Fase 4](fase-04-movimientos.md) dejamos registrado cada movimiento, primero con una llamada directa. Ese registro es la pieza que en CaribeConf se convierte en un evento.
