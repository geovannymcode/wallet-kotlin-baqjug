# Fase 2 · Exponer el saldo por REST

**Rama**: `fase-2`
**Lo que vas a lograr**: consultar el saldo de una cuenta con un `GET`, separando bien la vitrina (`api`) de la bodega (`internal`).

---

## Parte 1 — El DTO de respuesta (5 min)

Lo primero es decidir qué le mostramos al mundo. No devolvemos la entidad `Account` directo: devolvemos un objeto pensado para la respuesta, un DTO. Va en la `api` de la feature, porque es parte de la vitrina.

```kotlin title="account/api/AccountResponse.kt"
package com.baqjug.wallet.account.api

import java.math.BigDecimal
import java.util.UUID

data class AccountResponse(
    val id: UUID,
    val owner: String,
    val balance: BigDecimal
)
```

!!! abstract "Kotlin al paso: `data class`"
    Una `data class` es una clase pensada para llevar datos. Kotlin te genera gratis `equals`, `hashCode`, `toString` y `copy`. En una línea tienes un objeto inmutable con tres campos. En Java esto serían 40 líneas o un `record`.

!!! abstract "Concepto al paso: qué es un DTO y por qué no exponer la entidad"
    Un DTO (Data Transfer Object) es un objeto que existe solo para mover datos entre capas, en este caso hacia afuera por la API. Si devolvieras la entidad `Account` directo, tu API quedaría amarrada a cómo guardas los datos: cambiar la tabla te cambiaría el contrato REST. Con un DTO, la forma de la respuesta la decides tú, aparte de la base.

## Parte 2 — La interfaz del servicio, en la vitrina (5 min)

Otras features (como `transfer`, en la próxima fase) van a necesitar buscar cuentas. Para no acoplarlas a la implementación, exponemos una interfaz en la `api`.

```kotlin title="account/api/AccountService.kt"
package com.baqjug.wallet.account.api

import java.util.UUID

interface AccountService {
    fun getById(id: UUID): AccountResponse
}
```

!!! abstract "Concepto al paso: por qué una interfaz en vez de la clase directa"
    La interfaz es el contrato público de la feature. Quien la use depende del contrato, no de los detalles. Mañana puedes cambiar por completo cómo `account` guarda o busca las cuentas, y mientras respetes la interfaz, nadie afuera se entera ni se rompe.

## Parte 3 — La implementación, en la bodega (10 min)

La implementación vive en `internal`. Busca la cuenta y la traduce a DTO.

```kotlin title="account/internal/DefaultAccountService.kt"
package com.baqjug.wallet.account.internal

import com.baqjug.wallet.account.api.AccountResponse
import com.baqjug.wallet.account.api.AccountService
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.util.UUID

@Service
@Transactional(readOnly = true)
class DefaultAccountService(
    private val repository: AccountRepository
) : AccountService {

    override fun getById(id: UUID): AccountResponse {
        val account = repository.findById(id)
            .orElseThrow { AccountNotFoundException(id) }
        return AccountResponse(account.id, account.owner, account.balance)
    }
}

class AccountNotFoundException(id: UUID) :
    RuntimeException("No existe la cuenta $id")
```

!!! abstract "Spring al paso: `@Service` e inyección por constructor"
    `@Service` marca esta clase como un bean de lógica de negocio: Spring la crea y la administra. El `AccountRepository` que pide el constructor lo inyecta Spring automáticamente. En Kotlin, poner la dependencia como parámetro del constructor con `private val` es todo lo que necesitas. Nada de `@Autowired`.

!!! abstract "Spring al paso: `@Transactional(readOnly = true)`"
    `@Transactional` envuelve el método en una transacción de base de datos. `readOnly = true` le avisa que solo lee, lo que permite optimizaciones. En la próxima fase vas a ver `@Transactional` sin `readOnly`, para cuando sí escribimos.

!!! abstract "Kotlin al paso: `orElseThrow` y las lambdas"
    `findById` devuelve un `Optional`. `orElseThrow { ... }` dice: si no hay cuenta, lanza esta excepción. Lo que va entre llaves es una **lambda**, una función corta sin nombre. Cuando la lambda es el último argumento, Kotlin te deja sacarla del paréntesis, por eso queda `orElseThrow { ... }`.

## Parte 4 — El controlador REST (10 min)

El controlador es la puerta de entrada HTTP. Traduce una petición web a una llamada al servicio.

```kotlin title="account/internal/AccountController.kt"
package com.baqjug.wallet.account.internal

import com.baqjug.wallet.account.api.AccountResponse
import com.baqjug.wallet.account.api.AccountService
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import java.util.UUID

@RestController
@RequestMapping("/api/accounts")
class AccountController(
    private val accountService: AccountService
) {

    @GetMapping("/{id}")
    fun getById(@PathVariable id: UUID): AccountResponse =
        accountService.getById(id)
}
```

!!! abstract "Spring al paso: `@RestController`, `@RequestMapping`, `@GetMapping`, `@PathVariable`"
    `@RestController` dice que esta clase atiende peticiones HTTP y que lo que devuelva se serializa a JSON. `@RequestMapping("/api/accounts")` es el prefijo de ruta. `@GetMapping("/{id}")` atiende un `GET` a `/api/accounts/{id}`. `@PathVariable id` toma el `{id}` de la URL y te lo pasa como parámetro, ya convertido a `UUID`.

!!! note "El controlador inyecta la interfaz, no la implementación"
    Fíjate que el controlador pide `AccountService` (la interfaz de la `api`), no `DefaultAccountService`. Depende del contrato. Spring sabe cuál implementación inyectar porque solo hay una.

Convierte la excepción en un `404` con un manejador global:

```kotlin title="account/internal/AccountExceptionHandler.kt"
package com.baqjug.wallet.account.internal

import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class AccountExceptionHandler {

    @ExceptionHandler(AccountNotFoundException::class)
    fun handleNotFound(ex: AccountNotFoundException): ResponseEntity<Map<String, String>> =
        ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(mapOf("error" to (ex.message ?: "Cuenta no encontrada")))
}
```

!!! abstract "Spring al paso: `@RestControllerAdvice` y `@ExceptionHandler`"
    En vez de meter `try/catch` en cada controlador, `@RestControllerAdvice` centraliza el manejo de errores. `@ExceptionHandler(AccountNotFoundException::class)` dice: cuando alguien lance esa excepción, responde con esto. Acá traducimos "no existe la cuenta" a un HTTP `404` con un JSON de error.

## Parte 5 — Probarlo (5 min)

Arranca la app y consulta el saldo de Ana:

```bash
curl http://localhost:8080/api/accounts/11111111-1111-1111-1111-111111111111
```

```json
{ "id": "11111111-1111-1111-1111-111111111111", "owner": "Ana", "balance": 100000.00 }
```

Y una cuenta que no existe te da `404`.

## Cierre de la fase

```bash
./gradlew compileKotlin
git add .
git commit -m "fase-2: consulta de saldo por REST con separación api/internal"
git branch fase-2
```

Ya lees el saldo desde afuera. En la [Fase 3](fase-03-transferencia.md) viene lo bueno: mover plata de una cuenta a otra validando el saldo, dentro de una transacción.
