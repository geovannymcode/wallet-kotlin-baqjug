# Apéndice · Coroutines con Reactor y Arrow (runbook)

Este apéndice profundiza dos herramientas que asoman en la [Fase 9](fase-09-coroutines.md) cuando llevas coroutines a un proyecto real sobre Spring WebFlux (Netty): el **puente con Reactor** (`kotlinx-coroutines-reactor`) y **`parMap`** (`arrow-fx-coroutines`). No es parte del hilo del taller; es un runbook para cuando vayas a portar el patrón a tu propio código.

!!! note "El punto de partida"
    Coroutines **no reemplaza** Reactor: se apoya en él. Necesitas `spring-boot-starter-webflux` + Netty debajo. Tus `WebClient` (o cualquier `Mono`/`Flux` de Spring Data R2DBC, etc.) siguen siendo 100% reactivos por dentro; lo que cambia es el punto donde tú te enganchas a ese flujo.

## 1. `kotlinx-coroutines-reactor` — el puente

### Consumir un `Mono`/`Flux` desde una coroutine

```kotlin
suspend fun <T> Mono<T>.awaitSingle(): T          // NoSuchElementException si está vacío
suspend fun <T> Mono<T>.awaitSingleOrNull(): T?   // null si está vacío
suspend fun <T> Flux<T>.awaitFirst(): T
suspend fun <T> Flux<T>.awaitFirstOrNull(): T?
suspend fun <T> Flux<T>.awaitLast(): T
```

!!! warning "`awaitBody()` es de Spring, no del puente"
    `awaitBody()` / `awaitBodyOrNull()` son extensiones de **Spring** (`WebClientExtensions.kt`, paquete `org.springframework.web.reactive.function.client`), hechas para el `WebClient`. Por dentro llaman a `awaitSingle()` / `awaitSingleOrNull()`. Distinción útil: para un `Mono` que **no** viene de WebClient (R2DBC, uno armado a mano), no hay extensión de Spring — usas `awaitSingle()` directo.

### Convertir `Flux` ↔ `Flow`

```kotlin
fun <T> Flux<T>.asFlow(): Flow<T>
fun <T> Flow<T>.asFlux(): Flux<T>
```

Relevante si quieres exponer streams: Spring soporta `Flow<T>` como tipo de retorno directo en un handler Kotlin, en vez de `Flux<T>`.

### El camino inverso: los builders `mono { }` / `flux { }`

Cuando necesitas **producir** un `Mono`/`Flux` a partir de código `suspend` (por ejemplo, un `@Bean` reactivo, o un API que otro código reactivo consumirá):

```kotlin
fun endpoint(id: String): Mono<Book> = mono {
    val metadata = metadataClient.getMetadata(id)   // suspend, dentro del builder
    Book.from(metadata, reviewClient.getReviews(id))
}
```

### Propagación de contexto — lo que más se pasa por alto

Reactor lleva su propio `Context` (para `ReactiveSecurityContextHolder`, tracing de Micrometer, etc.). Ese contexto **solo** se inyecta en tu `CoroutineContext` dentro de `mono { }` / `flux { }`. Fuera de esos builders no lo tienes, salvo que el framework te lo propague (WebFlux sí lo hace para el `SecurityContext` en handlers `suspend`, desde Spring Security 5.4+). Por eso, si dentro de un handler `suspend` necesitas el usuario autenticado:

```kotlin
suspend fun getBook(...): Book {
    val auth = ReactiveSecurityContextHolder.getContext().awaitSingle().authentication
    // ...
}
```

No lo asumas "propagado"; léelo explícito con `awaitSingle()`.

### Errores y cancelación

Si una `suspend fun` lanza dentro de un `mono { }`, el `Mono` emite `onError` con esa excepción (menos `CancellationException`, que cancela sin error). Y si cancelas la coroutine (el cliente cerró la conexión y Spring cancela el `Job` del handler), la suscripción Reactor subyacente se cancela también, y viceversa. Esa es la cancelación "gratis".

## 2. `arrow-fx-coroutines` — `parMap`

Firma real (paquete `arrow.fx.coroutines`):

```kotlin
suspend fun <A, B> Iterable<A>.parMap(
    context: CoroutineContext = EmptyCoroutineContext,
    f: suspend CoroutineScope.(A) -> B
): List<B>

suspend fun <A, B> Iterable<A>.parMap(
    context: CoroutineContext = EmptyCoroutineContext,
    concurrency: Int,
    f: suspend CoroutineScope.(A) -> B
): List<B>
```

- Sin `concurrency`, lanza **todos** los elementos a la vez. El overload con `concurrency: Int` mete un semáforo interno y acota cuántas coroutines corren en paralelo.
- **Concurrencia estructurada real**: por dentro usa `coroutineScope`, así que si un elemento falla, cancela al resto y propaga (fail-fast) — igual que `async` + `awaitAll` manual.
- **`parMapOrAccumulate`**: en vez de fail-fast, acumula errores (con `Either`/`NonEmptyList` del DSL `Raise`). Útil para "3 de 20 fallaron" en lugar de tumbar toda la respuesta por uno.

## 3. La decisión de dependencia

| Necesidad | Sin Arrow (stdlib) | Con Arrow |
|-----------|--------------------|-----------|
| Paralelizar N, sin límite | `list.map { async { f(it) } }.awaitAll()` | `list.parMap { f(it) }` |
| Paralelizar N, con límite | `list.asFlow().flatMapMerge(N) { flow { emit(f(it)) } }.toList()` | `list.parMap(concurrency = N) { f(it) }` |
| Acumular errores (no fail-fast) | a mano (try/catch por elemento) | `parMapOrAccumulate` |

!!! danger "Regla honesta"
    Si el único uso de Arrow va a ser `parMap`, `flatMapMerge` de `kotlinx-coroutines-core` (que ya viene con WebFlux) te da concurrencia acotada **sin dependencia nueva** — esa es la opción por defecto. Arrow se justifica si ya usas o vas a usar su ecosistema (`Either`, `Raise`, validación de dominio): ahí `parMapOrAccumulate` deja de ser una dependencia aislada para una sola función.

## Ejercicios (sandbox: el consumer de la Fase 9)

Usa el `NotificationService` de la Fase 9 (correo, push, antifraude) como campo de pruebas, sin tocar tu proyecto real.

1. **Adaptar un cliente bloqueante.** Toma un cliente con `RestClient` (bloqueante) y conviértelo a `suspend` con `WebClient` + `awaitBody()`. No reescribas la config (timeouts, filters); solo cambia el punto de suspensión.
2. **Dos llamadas concurrentes.** Escribe `notificar(evento)` que llame a `email` y `push` en paralelo con `coroutineScope { async {} }` + `awaitAll`. Verifica que si `push` lanza, `email` se cancela.
3. **N llamadas con límite (stdlib).** Simula un lote de 50 movimientos; por cada uno llama a un servicio externo. Acótalo a 4 en paralelo con `asFlow().flatMapMerge(4) { … }.toList()`.
4. **N llamadas con Arrow.** Reescribe el ejercicio 3 con `parMap(concurrency = 4)`. Compara legibilidad.
5. **Acumular errores.** Con `parMapOrAccumulate`, haz que 2 de los 50 fallen y responde cuántos fallaron sin perder los 48 buenos.
6. **Contexto reactivo.** En un handler `suspend` de WebFlux, lee el `SecurityContext` con `ReactiveSecurityContextHolder.getContext().awaitSingle()`. Comprueba que fuera de `mono { }` no está propagado si no lo lees explícito.
7. **Cancelación.** Cancela el `Job` del handler a mitad de una llamada y verifica que la suscripción Reactor subyacente se cancela.

## Prompt de implementación para tu proyecto

Ajusta los corchetes y pégalo cuando vayas a migrar:

> Quiero introducir Kotlin coroutines en **[nombre del servicio/módulo]** sobre Spring WebFlux (Netty), reemplazando **[llamadas bloqueantes con RestClient / cadenas Mono-Flux existentes]** en **[rutas/archivos concretos]**.
>
> **Contexto del proyecto:**
> - Stack: **[Spring Boot version]**, Kotlin **[version]**, **[servlet/webflux actual]**
> - Clientes HTTP salientes: **[WebClient/RestClient hacia qué servicios]**
> - Endpoints a migrar: **[listar rutas/controllers]**
> - ¿Ya usamos Arrow para algo más (Either, Raise, validación)? **[sí/no]**
>
> **Requisitos:**
> 1. Los handlers de controller deben quedar como funciones `suspend`, no envueltas manualmente en `Mono`.
> 2. No reescribir los `WebClient` existentes: adaptar con `awaitBody()`/`awaitBodyOrNull()`/`awaitSingleOrNull()` según corresponda, manteniendo la config actual (timeouts, filters, interceptors).
> 3. Llamadas independientes conocidas (2–3) → `coroutineScope { async {} }` + `awaitAll`, para concurrencia estructurada y cancelación automática si una falla.
> 4. Llamadas sobre una colección (N elementos) → evaluar entre: (a) `list.asFlow().flatMapMerge(concurrency = N) { … }.toList()` (sin dependencia nueva) o (b) `list.parMap(concurrency = N) { … }` (si ya usamos o vamos a adoptar Arrow). Justifica cuál eliges antes de implementar.
> 5. Si necesito leer el `SecurityContext` reactivo dentro de un handler `suspend`, usar `awaitSingle()` explícito, no asumir propagación automática fuera de `mono { }` / `flux { }`.
> 6. No tragar excepciones: los errores se propagan igual que en la versión actual (mismos status codes / `ResponseStatusException`).
> 7. Diff archivo por archivo, sin reescribir lo que no cambia. Si algún `WebClient` no tiene timeout, dilo aparte como hallazgo; no lo arregles sin que te lo pida.
>
> No introduzcas `arrow-fx-coroutines` si el único uso va a ser `parMap` y `flatMapMerge` cubre el caso — pregúntame primero.

Para la teoría base de coroutines (qué es `suspend`, `coroutineScope`, dispatchers), vuelve a la [Fase 9](fase-09-coroutines.md).
