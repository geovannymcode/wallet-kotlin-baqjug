# Fase 10 · Despliegue a la nube

**Rama**: opcional, es el paso de "shipping". Hasta acá corriste todo en tu máquina; esta fase lleva la wallet a la nube y hace que cada `push` a la rama conectada la construya, la teste y la despliegue sola.
**Lo que vas a lograr**: empaquetar la app en una imagen Docker, montar un pipeline de GitHub Actions que compila y testea, y desplegar a Render con la **configuración real** que pide un entorno de nube: un puerto dinámico, un broker Kafka administrado (con sus ACLs y su topic de DLQ), y el correo por **API HTTP** —porque el SMTP saliente suele venir bloqueado—.

!!! warning "Esta fase es de operaciones (ops)"
    El código de **negocio** no cambia. Lo único que se ajusta es la plomería para que la wallet viva fuera de tu laptop: imagen Docker, integración continua, y unas cuantas piezas de configuración que en local no se notan pero en la nube son la diferencia entre "arranca" y "no arranca". Es opcional, pero es lo que separa "me corre en local" de "está en internet".

!!! note "¿Y correr todo esto local con Docker?"
    Esta fase es el camino a la **nube**. Si lo que quieres es levantar el stack completo (Postgres, Redpanda y Mailpit) en tu máquina con `docker compose`, eso vive aparte, en la [guía de configuración](configuracion.md#parte-1-entorno-local-con-docker-compose). La app está hecha para **ambos** casos; lo que cambia es la configuración por perfil.

---

## Parte 1 — Empaquetar la app en una imagen Docker

Para que la wallet corra en cualquier lado igual que en tu máquina, la metemos en una **imagen Docker**: un paquete con la app y todo lo que necesita para arrancar.

!!! abstract "Concepto al paso: imagen Docker y build multi-etapa"
    Una imagen Docker es una plantilla con tu app lista para correr. La construimos en **dos etapas**: la primera tiene el JDK completo y compila el `jar`; la segunda, mucho más liviana, solo tiene el JRE y el `jar` ya compilado. Así la imagen final no carga con el compilador ni el código fuente: pesa menos y arranca más rápido.

```dockerfile title="Dockerfile"
# Etapa 1: compilar
FROM eclipse-temurin:25-jdk AS build
WORKDIR /app
COPY . .
RUN ./gradlew bootJar --no-daemon

# Etapa 2: correr (imagen liviana, solo JRE)
FROM eclipse-temurin:25-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

!!! note "Alternativa sin Dockerfile"
    Spring Boot puede armar la imagen solo, con `./gradlew bootBuildImage` (usa buildpacks, sin escribir un Dockerfile). Cualquiera de las dos sirve; el Dockerfile es más explícito y portable entre hosts, por eso lo mostramos.

## Parte 2 — El pipeline de integración continua (GitHub Actions)

Queremos que cada cambio se compile y se testee solo, antes de desplegar nada.

!!! abstract "Concepto al paso: CI/CD y GitHub Actions"
    **CI** (integración continua) es automatizar que, con cada `push`, se compile y se corran los tests. **CD** (despliegue continuo) es que, si eso pasa, se despliegue solo. **GitHub Actions** ejecuta esos pasos en un *runner* (una máquina que GitHub te presta) según recetas en `.github/workflows/`. Cada receta es un *workflow*.

```yaml title=".github/workflows/ci.yml"
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configurar JDK 25
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '25'

      - name: Compilar y testear
        run: ./gradlew test
```

!!! abstract "Spring al paso: por qué los tests corren sin infra"
    ¿Te acuerdas del test de la Fase 4 con MockK? Corre sin base ni broker, así que el runner de GitHub lo ejecuta sin levantar Supabase ni Redpanda. Los tests que sí necesitan infra real (con Testcontainers) también corren en Actions, porque el runner tiene Docker; eso ya es un paso más avanzado.

## Parte 3 — Desplegar a Render

Render despliega tu imagen Docker directo desde el repo. La forma más simple para un taller es dejar que **el propio host construya y despliegue** con cada `push` a la rama que conectes:

1. En Render, creas un **Web Service** y conectas tu repo de GitHub.
2. Eliges **Docker** como entorno: Render detecta tu `Dockerfile` y construye la imagen.
3. En **Environment**, pones las variables (las de la [tabla de más abajo](#parte-5-variables-de-entorno-y-secretos)).
4. Cada `push` a la rama trackeada dispara un build y un deploy automáticos.

Así queda un reparto limpio: **GitHub Actions hace el CI** (compila y testea) y **Render hace el CD** (construye la imagen y la despliega).

!!! warning "‘Restart’ no es lo mismo que ‘Deploy’"
    En Render, **Restart** solo reinicia el contenedor con la **misma imagen ya construida** —no baja código nuevo ni reconstruye el `Dockerfile`—. El único evento que recoge tu último commit es **Deploy** (verás *"Deploy live for `<sha>`"* en la pestaña *Events*). Si aplicaste un fix y "el error sigue igual", revisa primero **qué commit quedó desplegado** en *Events* antes de dudar del fix. Y ojo con la rama: Render despliega la que **conectaste**, que no tiene por qué ser `main`.

!!! warning "La UI de estos hosts cambia seguido"
    Los nombres de botones, la capa gratis y los límites de Render cambian con el tiempo. Toma esto como el mapa —conectar repo, elegir Docker, poner variables, deploy con push— y confirma el detalle en su documentación al momento de hacerlo.

## Parte 4 — La configuración que pide un entorno real

Esto es lo que en local no se nota y en la nube sí. Cada pieza responde a una diferencia concreta entre tu máquina y un host administrado.

### El puerto lo pone el host

Render (y casi cualquier PaaS) te asigna el puerto por la variable `PORT` y espera que el proceso escuche **ahí**, no en un `8080` fijo. Si Tomcat arranca en 8080 y el host esperaba otro puerto, verás en el log `No open ports detected` en bucle aunque la app haya arrancado bien.

```yaml title="src/main/resources/application.yaml"
server:
  port: ${PORT:8080}
```

El *fallback* a `8080` mantiene el comportamiento local (sin `PORT` definido, sigue en 8080). No configuras `PORT` a mano en Render: lo inyecta el host.

### Kafka contra un broker administrado (Redpanda Cloud)

En local, el broker es tuyo y arranca con todo listo. En la nube, el broker es un servicio aparte con **autenticación, ACLs y topics** que quizá no existen aún cuando la app enciende. Tres ajustes cubren eso:

**1. Que un fallo de autorización al arrancar no mate al consumidor para siempre.** Por defecto, Spring Kafka trata un `AuthorizationException` como **fatal** y **detiene** el consumidor de forma permanente; arreglar la ACL después no lo revive, toca reiniciar el servicio a mano. Con un intervalo de reintento, en cambio, reintenta hasta que la ACL o el topic existan:

```yaml title="src/main/resources/application.yaml (Kafka, resiliencia)"
spring:
  kafka:
    listener:
      auth-exception-retry-interval: 10s
```

**2. El topic de la DLQ tiene que existir (y con permisos).** En la [Fase 8](fase-08-que-sigue.md) el `DeadLetterPublishingRecoverer` publica los muertos en `wallet.movements-dlt`. En un broker administrado normalmente **no puedes auto-crear** topics, así que créalo tú en la consola (mismas particiones que `wallet.movements`) y dale acceso al usuario SASL a **los dos** nombres. Si no existe o falta el permiso, Kafka responde `TopicAuthorizationException` —y como la publicación a la DLQ nunca tiene éxito, el consumidor reintenta en bucle y **traba la partición** para todos—.

!!! danger "‘Not authorized to access topics: [wallet.movements-dlt]’ en bucle"
    Kafka responde `TOPIC_AUTHORIZATION_FAILED` tanto cuando **no tienes permiso** como cuando el **topic no existe** (es a propósito: no te filtra qué topics hay). Si ves ese error repitiéndose cada segundo, la causa casi siempre es que el topic de la DLQ no está creado o el usuario SASL no lo cubre. Créalo y concede el acceso, y el bucle se corta.

**3. El módulo Kotlin de Jackson tiene que ser el de Jackson 2.** El `JsonSerializer`/`JsonDeserializer` de spring-kafka usan **Jackson 2** (`com.fasterxml.jackson`). Si agregas el módulo Kotlin de **Jackson 3** (`tools.jackson.module`), nunca se registra en ese `ObjectMapper` y deserializar un `data class` de Kotlin falla con `InvalidDefinitionException: no Creators`. Usa el módulo de Jackson 2:

```kotlin title="build.gradle.kts (dependencies)"
implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
```

### El correo en producción: del SMTP directo a una API HTTP

En la [Fase 7](fase-07-consumir-evento.md) `notification` manda el correo por **SMTP** contra Mailpit. Eso es perfecto en local, pero en la nube choca con un muro: **muchos hosts bloquean el SMTP saliente**. Render, por ejemplo, bloquea los puertos SMTP (25, 465 y 587) en los Web Services del plan Free desde septiembre de 2025.

La salida es no atarse a un transporte: abstraemos el envío detrás de un **puerto** (`MailPort`) con dos implementaciones que Spring elige **por perfil**. En local, SMTP contra Mailpit; en la nube, la **API HTTP** de un proveedor (acá Resend), que va por HTTPS (443) y no está bloqueada.

```kotlin title="notification/domain/MailPort.kt"
package com.baqjug.wallet.notification.domain

data class Mail(val to: String, val subject: String, val body: String)

interface MailPort {
    fun send(mail: Mail)
}
```

```kotlin title="notification/domain/SmtpMailPort.kt (perfil local)"
@Component
@Profile("local")
class SmtpMailPort(
    private val mailSender: JavaMailSender,
    @Value("\${wallet.mail.from}") private val from: String
) : MailPort {
    override fun send(mail: Mail) {
        val msg = SimpleMailMessage().apply {
            this.from = from
            setTo(mail.to)
            subject = mail.subject
            text = mail.body
        }
        mailSender.send(msg)
    }
}
```

```kotlin title="notification/domain/ResendMailPort.kt (perfil de nube)"
@Component
@Profile("!local")
class ResendMailPort(
    @Value("\${wallet.mail.resend-api-key}") private val apiKey: String,
    @Value("\${wallet.mail.from}") private val from: String
) : MailPort {

    private val client = RestClient.create("https://api.resend.com")

    override fun send(mail: Mail) {
        client.post()
            .uri("/emails")
            .header("Authorization", "Bearer $apiKey")
            .contentType(MediaType.APPLICATION_JSON)
            .body(mapOf(
                "from" to from,
                "to" to mail.to,
                "subject" to mail.subject,
                "text" to mail.body
            ))
            .retrieve()
            .toBodilessEntity()
    }
}
```

`EmailNotifier` (y el `DeadLetterListener` de la Fase 8, si lo usas para alertar) pasan a depender de `MailPort` en vez de `JavaMailSender` directamente. No saben —ni les importa— si detrás hay SMTP o HTTP.

!!! abstract "Spring al paso: `@Profile` elige la implementación"
    `@Profile("local")` monta ese bean **solo** cuando el perfil `local` está activo; `@Profile("!local")`, cuando **no** lo está. Como ambos implementan `MailPort`, `EmailNotifier` recibe el que corresponda sin cambiar una línea. En tu máquina activas el perfil con `SPRING_PROFILES_ACTIVE=local` (SMTP + Mailpit); en Render lo dejas sin ese perfil (gana `ResendMailPort`, la API HTTP). El mismo binario, dos transportes.

!!! warning "SMTP saliente bloqueado en planes Free"
    No es un problema tuyo de configuración: es política del host. Render bloquea 25/465/587 en Free desde sep. 2025 ([changelog oficial](https://render.com/changelog/free-web-services-will-no-longer-allow-outbound-traffic-to-smtp-ports)). Por eso en producción vamos por HTTPS a la API del proveedor. Si tu host sí permite SMTP, puedes seguir con `JavaMailSender` y un SMTP real; la abstracción `MailPort` te deja elegir sin reescribir `notification`.

!!! note "El remitente y el destinatario, en modo sandbox"
    Sin un dominio propio verificado, Resend trabaja en **sandbox**: el remitente debe ser `onboarding@resend.dev` (un `MAIL_FROM` con dominio inválido como `wallet@localhost` da `422`), y solo entrega al **correo registrado** en tu cuenta de Resend. Como `EmailNotifier` escribe al email de la cuenta **destino** (el dato semilla `V1__create_accounts.sql`), en sandbox ese correo debe ser el tuyo registrado —actualízalo en la base, no en la migración compartida—. Para mandar a cualquier destinatario, verifica un dominio propio en Resend y sales del sandbox.

## Parte 5 — Variables de entorno y secretos

El código nunca sabe un secreto; lo lee del entorno. Lo que cambia es **quién** pone esas variables en cada lado:

- **En tu máquina**: en IntelliJ (**Run → Edit Configurations → Environment variables**), o en el `docker compose` del [entorno local](configuracion.md#parte-1-entorno-local-con-docker-compose).
- **En GitHub Actions**: en **Settings → Secrets and variables → Actions** del repo.
- **En Render**: en la sección **Environment** del servicio.

Las que necesita la wallet **en la nube**:

| Variable | Para qué |
|----------|----------|
| `SUPABASE_DB_URL` / `SUPABASE_DB_USER` / `SUPABASE_DB_PASSWORD` | La base Postgres (Supabase) |
| `REDPANDA_BOOTSTRAP` / `REDPANDA_USER` / `REDPANDA_PASSWORD` | El broker de eventos (Redpanda Cloud) |
| `RESEND_API_KEY` | La API key de Resend (autenticación `Bearer` sobre HTTPS) |
| `MAIL_FROM` | El remitente (`onboarding@resend.dev` en sandbox) |
| `MAIL_OPS_TO` | Destino de las alertas de DLQ (en sandbox, tu correo registrado en Resend) |
| `JAVA_TOOL_OPTIONS` | Para que la JVM quepa en los 512 MB del plan Free (p. ej. `-Xmx400m`) |
| `WALLET_DEMO_DLQ_SENTINEL` | Flag de la demo de DLQ — apagado (`false`) fuera de la demostración |

!!! note "`PORT` no la pones tú"
    Render inyecta `PORT` solo; tu app la lee con `server.port: ${PORT:8080}`. Y como el correo en la nube va por HTTP (Resend), **no** necesitas las `SMTP_*` en producción: esas viven solo en el [entorno local](configuracion.md#parte-1-entorno-local-con-docker-compose).

## Cierre de la fase

Con esto, la wallet dejó de vivir solo en tu laptop: cada `push` pasa por CI (compila y testea), se construye la imagen Docker, y Render la despliega. Corre en la nube, con su base en Supabase, sus eventos en Redpanda Cloud, y correos que salen por la API de Resend —mientras que en local siguen cayendo en Mailpit, sin tocar el código—.

Y con eso cerramos el taller: arrancaste con un endpoint REST contra una base, y terminaste con una wallet transaccional, desacoplada por eventos, endurecida con outbox/idempotencia/DLQ, testeada, empaquetada y desplegada. Nada de humo: código y tuberías de verdad.

Para levantar todo local con un solo `docker compose`, sigue en la [guía de configuración](configuracion.md). Y revisa las [Referencias](referencias.md) para seguir por tu cuenta.
