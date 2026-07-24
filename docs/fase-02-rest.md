# Fase 2 · Exponer el saldo por REST

**Rama**: `fase-2`
**Lo que vas a lograr**: consultar el saldo de una cuenta con un `GET`, separando la lógica de negocio (`domain`) de la puerta HTTP (`web`).

---

## Parte 1 — El DTO de respuesta

Lo primero es decidir qué le mostramos al mundo. No devolvemos la entidad `AccountEntity` directo: devolvemos un objeto pensado para la respuesta, un DTO. Va en `domain`, junto al resto de la lógica de la feature.

```kotlin title="account/domain/AccountResponse.kt"
package com.baqjug.wallet.account.domain

import java.math.BigDecimal
import java.util.UUID

data class AccountResponse(
    val id: UUID,
    val owner: String,
    val email: String,
    val balance: BigDecimal
)
```

!!! abstract "Kotlin al paso: `data class`"
    Una `data class` es una clase pensada para llevar datos. Kotlin te genera gratis `equals`, `hashCode`, `toString` y `copy`. En una línea tienes un objeto inmutable con tres campos. En Java esto serían 40 líneas o un `record`.

!!! abstract "Concepto al paso: qué es un DTO y por qué no exponer la entidad"
    Un DTO (Data Transfer Object) es un objeto que existe solo para mover datos entre capas, en este caso hacia afuera por la API. Si devolvieras la entidad `AccountEntity` directo, tu API quedaría amarrada a cómo guardas los datos: cambiar la tabla te cambiaría el contrato REST. Con un DTO, la forma de la respuesta la decides tú, aparte de la base.

## Parte 2 — El mapper: de la entidad al DTO

Tenemos dos objetos que se parecen pero cumplen papeles distintos: `AccountEntity`, que es cómo se guarda la cuenta en la base, y `AccountResponse`, que es cómo la mostramos por la API. Necesitamos algo que traduzca de uno al otro. Ese algo es un **mapper**.

```kotlin title="account/domain/AccountMapper.kt"
package com.baqjug.wallet.account.domain

object AccountMapper {
    fun toResponse(entity: AccountEntity) = AccountResponse(
        id = entity.id,
        owner = entity.owner,
        email = entity.email,
        balance = entity.balance
    )
}
```

!!! abstract "Kotlin al paso: `object`"
    `object` declara un **singleton**: una única instancia para toda la app, sin que tú la crees en ningún lado. Como el mapper no guarda estado (solo transforma datos de entrada en datos de salida), no tiene sentido tener muchas copias. `AccountMapper.toResponse(...)` se llama directo, como un método estático de Java.

!!! note "¿No es trabajo de más este mapper?"
    A primera vista sí. Pero fíjate qué te compra: `AccountEntity` puede cambiar (agregar una columna, renombrar un campo) sin que tu API se entere, porque lo que sale es `AccountResponse` y en el medio está el mapper. Sin esta separación, el día que tocas la tabla, cambias sin querer el JSON que tus clientes ya consumen. El mapper es barato; romper el contrato de tu API, caro.

!!! note "Acá se decide qué se muestra y qué no"
    Fíjate que el mapper copia `owner`, `email` y `balance`, pero **no** `createdAt` ni `updatedAt`. Esos campos de auditoría existen en la tabla, pero no los sacamos por la API. El mapper es justo el lugar donde eliges qué de la entidad se vuelve público.

## Parte 3 — El servicio, en el corazón de la feature

El servicio tiene la lógica: busca la cuenta y la entrega ya traducida a DTO. Vive en `domain`.

```kotlin title="account/domain/AccountService.kt"
package com.baqjug.wallet.account.domain

import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.util.UUID

@Service
@Transactional(readOnly = true)
class AccountService(
    private val repository: AccountRepository
) {

    fun getById(id: UUID): AccountResponse {
        val account = repository.findById(id)
            .orElseThrow { AccountNotFoundException(id) }
        return AccountMapper.toResponse(account)
    }
}
```

Y la excepción que lanza cuando no encuentra la cuenta, en su propio archivo:

```kotlin title="account/domain/AccountNotFoundException.kt"
package com.baqjug.wallet.account.domain

import java.util.UUID

class AccountNotFoundException(id: UUID) :
    RuntimeException("No existe la cuenta $id")
```

!!! danger "Ojo acá: el servicio es una clase, NO una interfaz"
    En muchos tutoriales verías una interfaz `AccountService` y una clase `AccountServiceImpl` implementándola, aunque solo exista una implementación. Nosotros no. `AccountService` es una **clase concreta y ya**. Como dijimos en la Fase 0: crear la interfaz "por si algún día hay otra implementación" es resolver un problema que casi nunca llega, y si llega, el IDE te la extrae en dos teclas. Menos archivos, menos ruido, mismo resultado.

!!! abstract "Spring al paso: `@Service` e inyección por constructor"
    `@Service` marca esta clase como un bean de lógica de negocio: Spring la crea y la administra. El `AccountRepository` que pide el constructor lo inyecta Spring solo. En Kotlin, poner la dependencia como parámetro del constructor con `private val` es todo lo que necesitas. Nada de `@Autowired`.

!!! abstract "Spring al paso: `@Transactional(readOnly = true)`"
    `@Transactional` envuelve el método en una transacción de base de datos. `readOnly = true` avisa que solo lee, lo que permite optimizaciones. En la próxima fase vas a ver `@Transactional` sin `readOnly`, para cuando sí escribimos.

!!! abstract "Kotlin al paso: `orElseThrow` y las lambdas"
    `findById` devuelve un `Optional`. `orElseThrow { ... }` dice: si no hay cuenta, lanza esta excepción. Lo que va entre llaves es una **lambda**, una función corta sin nombre. Cuando la lambda es el último argumento, Kotlin te deja sacarla del paréntesis, por eso queda `orElseThrow { ... }`.

## Parte 4 — El controlador REST, en la puerta `web`

El controlador es la puerta de entrada HTTP. Traduce una petición web en una llamada al servicio, y nada más. Vive en `web`, separado del `domain`.

```kotlin title="account/web/AccountController.kt"
package com.baqjug.wallet.account.web

import com.baqjug.wallet.account.domain.AccountResponse
import com.baqjug.wallet.account.domain.AccountService
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

!!! note "El controlador no tiene lógica, solo delega"
    Fíjate que el método del controlador no valida, no calcula, no toca la base: le pasa la pelota al `AccountService`. Esa es la regla de oro de la capa `web`: extraer los datos de la petición y delegar. Toda la lógica vive en `domain`. Así, el día que quieras disparar lo mismo desde otro lado (una tarea programada, un evento de Kafka), reusas el servicio sin arrastrar nada de HTTP.

Falta convertir la excepción en un `404`. Eso lo hace un manejador de errores **compartido por toda la app**, que ponemos en un paquete `web` de nivel superior (no dentro de una feature, porque sirve a todas):

```kotlin title="web/exception/GlobalExceptionHandler.kt"
package com.baqjug.wallet.web.exception

import com.baqjug.wallet.account.domain.AccountNotFoundException
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(AccountNotFoundException::class)
    fun handleNotFound(ex: AccountNotFoundException): ResponseEntity<Map<String, String>> =
        ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(mapOf("error" to (ex.message ?: "Cuenta no encontrada")))
}
```

!!! abstract "Spring al paso: `@RestControllerAdvice` y `@ExceptionHandler`"
    En vez de meter `try/catch` en cada controlador, `@RestControllerAdvice` centraliza el manejo de errores de toda la app en un solo lugar. `@ExceptionHandler(AccountNotFoundException::class)` dice: cuando alguien lance esa excepción, responde con esto. Acá traducimos "no existe la cuenta" a un HTTP `404` con un JSON de error. Como es global, cualquier feature que lance esa excepción obtiene el mismo `404`, sin repetir código.

## Parte 5 — Probarlo

Arranca la app y consulta el saldo de Elena:

```bash
curl http://localhost:8080/api/accounts/11111111-1111-1111-1111-111111111111
```

```json
{ "id": "11111111-1111-1111-1111-111111111111", "owner": "Elena", "email": "elena@example.com", "balance": 100000.00 }
```

Y una cuenta que no existe te da `404`.

## Cierre de la fase

```bash
./gradlew compileKotlin
git add .
git commit -m "fase-2: consulta de saldo por REST con separacion domain/web"
git branch fase-2
```

Ya lees el saldo desde afuera. En la [Fase 3](fase-03-transferencia.md) viene lo bueno: mover plata de una cuenta a otra validando el saldo, dentro de una transacción.
