# Punto de partida · Si arrancas por los eventos

**Bienvenido.** Vas a construir la parte más jugosa de una wallet: la que desacopla el sistema con **eventos**. Pero antes de escribir la primera línea, necesitas dos cosas para no perderte: **entender qué se construyó antes** —para que nada te suene a magia— y **tener el proyecto corriendo en tu máquina**. Esta página te da las dos, más un repaso rápido del Kotlin que vamos a usar. Cinco minutos aquí, y arrancamos todos parejos: sepas o no sepas Kotlin.

## Qué se construyó de la Fase 0 a la 4

Antes de los eventos, las primeras fases levantan una **wallet REST completa y funcionando**:

- **Consultar el saldo** de una cuenta por HTTP (`GET /api/accounts/{id}`).
- **Transferir plata** de una cuenta a otra (`POST /api/transfers`), validando que haya saldo, dentro de una **transacción** que nunca deja saldos rotos.
- **Registrar cada movimiento** en su propia tabla, con una **llamada directa** desde `transfer`.

Todo sobre Kotlin + Spring Boot, con arquitectura **por features** (Tomato: `domain`/`web`, servicios concretos, un mapper entre la entidad y el DTO), Postgres gestionado en Supabase y el esquema versionado con Flyway. Es una wallet que mueve plata de verdad, síncrona, contra una base real.

!!! abstract "La pieza clave para lo que viene"
    En la Fase 4, `transfer` registra el movimiento con una **llamada directa** a `movement`. Esa costura —esa llamada síncrona y acoplada— es justo lo que la parte de eventos va a cortar. Guárdala en la cabeza: es el "antes" de toda la segunda mitad.

## Por qué arrancamos aquí

La parte de eventos (Fases 5–10) **se apoya** en esa base REST; no la reinventa. Construir las Fases 0 a 4 en vivo se comería el tiempo que queremos dedicarle a lo nuevo: **por qué** desacoplar con eventos, y **cómo** publicarlos y consumirlos. Por eso no la escribimos desde cero hoy: te la damos **lista**, y arrancamos donde empieza lo interesante.

Y algo importante: aquí va a haber gente que **nunca ha tocado Kotlin**, y está perfecto. En las Fases 0 a 4, cada pieza del lenguaje se explicó la primera vez que apareció. Como esas fases te las saltas, la siguiente sección te deja ese repaso en una sola página: así, cuando veas un `data class`, una `sealed class` o un `interface`, ya sabes qué es y no te frena.

## La base de Kotlin que vas a necesitar

Un repaso exprés de las piezas del lenguaje que salieron de la Fase 0 a la 4 y que vas a seguir viendo. Si ya sabes Kotlin, sáltalo; si no, léelo con calma, es corto.

**`val` y `var` — inmutable vs. mutable.** `val` es un valor que **no cambia** después de asignarlo (como `final` en Java); `var` sí cambia. En la wallet, el `id` y el dueño de una cuenta son `val` (fijos); el saldo es `var` (sube y baja).

```kotlin
val id = UUID.randomUUID()      // fijo, nunca cambia
var balance = BigDecimal.ZERO   // cambia: el saldo sube y baja
```

**`class` y el constructor en una línea.** En Kotlin declaras la clase y su constructor juntos. Cada parámetro con `val`/`var` es además una propiedad de la clase. Nada de getters/setters ni de `this.x = x`.

```kotlin
class AccountEntity(val owner: String, var balance: BigDecimal)
```

**`data class` — una clase para llevar datos.** Kotlin te genera gratis `equals`, `hashCode`, `toString` y `copy`. En una línea tienes un objeto con sus campos; en Java serían 40 líneas.

```kotlin
data class AccountResponse(val id: UUID, val owner: String, val balance: BigDecimal)
```

**`interface` — un contrato.** Dice **qué** operaciones existen, no **cómo**. En la wallet, los repositorios son interfaces y Spring Data las implementa por ti.

```kotlin
interface AccountRepository : JpaRepository<AccountEntity, UUID>
```

**`sealed class` — una jerarquía cerrada.** Un conjunto **cerrado** de subclases: el compilador las conoce todas. Modela los finales posibles de una operación, y en un `when` te obliga a cubrir todos los casos (si falta uno, **no compila**). Es un `enum` con esteroides: cada caso puede llevar sus propios datos.

```kotlin
sealed class MoveResult {
    data object Success : MoveResult()
    data object InsufficientFunds : MoveResult()
    data class AccountNotFound(val id: UUID) : MoveResult()
}
```

**Y dos que verás seguido:**

- **Null-safety**: un tipo con `?` (`String?`) puede ser nulo; sin `?`, nunca lo es. El operador Elvis `?:` da un valor por defecto si algo viene nulo.
- **`when`**: como un `switch`, pero de verdad; sobre una `sealed class` te obliga a cubrir todos los casos.

!!! note "Y en Spring, las anotaciones"
    Verás `@Service` (un bean de lógica), `@RestController` (atiende HTTP y devuelve JSON), `@Entity` (una clase que se guarda en una tabla) y `@Transactional` (envuelve el método en una transacción). Cada una se explica cuando aparece; por ahora quédate con que **le dicen a Spring qué papel juega cada clase**.

## Ubícate: la rama y el proyecto

El **código** vive en un repo aparte de esta guía: [`github.com/geovannymcode/wallet`](https://github.com/geovannymcode/wallet). Para tener esa wallet REST completa sin escribirla, ponte en el estado exacto en el que quedó la Fase 4:

```bash
# 1. Clona el proyecto del código (repo aparte de esta guía)
git clone https://github.com/geovannymcode/wallet.git
cd wallet

# 2. Ubícate en la rama con la base REST ya lista (estado de la Fase 4)
git checkout fase-4
```

Configura las variables de entorno de Supabase (las de [Requisitos y setup](requisitos.md)) y levanta la app desde IntelliJ. Deberías ver a Flyway aplicar las migraciones y la app quedar escuchando.

!!! tip "Por qué en esta rama y no en `main`"
    Cada fase del taller es una rama. `fase-4` deja el proyecto justo con la wallet REST terminada —entidad, transferencia transaccional, registro de movimientos y tests— **sin** la parte de eventos encima. Es la **línea de salida** limpia para lo que vamos a construir. Si te pierdes en cualquier momento, vuelves a `git checkout fase-4` y arrancas de nuevo desde un estado conocido.

## Verifica que la base corre

Antes de meterle eventos, comprueba que la wallet síncrona funciona. Haz una transferencia:

```bash
curl -X POST http://localhost:8080/api/transfers \
  -H "Content-Type: application/json" \
  -d '{
        "fromId": "11111111-1111-1111-1111-111111111111",
        "toId": "22222222-2222-2222-2222-222222222222",
        "amount": 15000.00
      }'
```

Si te responde `200 COMPLETED`, y en Supabase ves el saldo movido y una fila nueva en `movements`, estás listo.

## Ya con contexto, sigue

Tienes la base corriendo y sabes qué hace. Ahora sí: la [Fase 5](fase-05-por-que-eventos.md) te explica **por qué** esa llamada directa se vuelve un problema cuando el negocio crece, y de la [Fase 6](fase-06-publicar-evento.md) en adelante la conviertes en un **evento** que otros servicios consumen sin que `transfer` se entere.
