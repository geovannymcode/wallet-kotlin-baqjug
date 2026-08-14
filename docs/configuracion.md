# Guía de configuración

## Entorno local, Redpanda Cloud y CI

Esta guía cubre tres piezas que sostienen el taller y que conviene tener claras antes de la [Fase 6](fase-06-publicar-evento.md) (eventos) y la [Fase 10](fase-10-despliegue.md) (despliegue):

1. **Entorno local con Docker Compose** — Postgres, Redpanda y Mailpit en tu máquina.
2. **Redpanda Cloud (Serverless)** — el broker en la nube, con sus usuarios y ACLs.
3. **Integración continua con GitHub Actions** — que cada `push` compile y testee solo.

Puedes hacer el taller entero con la opción 1 (todo local) o con la 2 (broker en la nube). La 3 es independiente de las otras dos.

---

## Parte 1 — Entorno local con Docker Compose

### Concepto al paso: por qué Compose

Levantar Postgres, un broker y un servidor de correo a mano es un dolor. Docker Compose describe los servicios en un archivo y los levanta con un comando. Cuando termines, `docker compose down -v` los borra sin dejar rastro en tu máquina.

### El archivo

```yaml title="docker-compose.yml"
services:
  # --- Base de datos ---
  postgres:
    image: postgres:17
    container_name: wallet-postgres
    environment:
      POSTGRES_DB: wallet
      POSTGRES_USER: wallet
      POSTGRES_PASSWORD: wallet
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U wallet -d wallet"]
      interval: 5s
      retries: 10

  # --- Broker de eventos (compatible con el protocolo de Kafka) ---
  redpanda:
    image: redpandadata/redpanda:v25.1.1
    container_name: wallet-redpanda
    command:
      - redpanda
      - start
      - --smp=1
      - --overprovisioned
      - --node-id=0
      - --check=false
      - --kafka-addr=internal://0.0.0.0:9092,external://0.0.0.0:19092
      - --advertise-kafka-addr=internal://redpanda:9092,external://localhost:19092
    ports:
      - "19092:19092"   # el puerto al que se conecta tu app desde el host
      - "9644:9644"     # API de administración

  # --- Consola web de Redpanda: para ver topics y mensajes ---
  console:
    image: redpandadata/console:latest
    container_name: wallet-redpanda-console
    environment:
      KAFKA_BROKERS: redpanda:9092
    ports:
      - "8081:8080"
    depends_on:
      - redpanda

  # --- Servidor de correo de mentira: atrapa todo lo que la app envíe ---
  mailpit:
    image: axllent/mailpit:latest
    container_name: wallet-mailpit
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # bandeja web
```

### Levantarlo

```bash
docker compose up -d
docker compose ps        # los cuatro servicios deben estar "running"
```

Direcciones útiles:

| Servicio | Dónde |
|---|---|
| Postgres | `localhost:5432` (usuario, clave y base: `wallet`) |
| Redpanda | `localhost:19092` |
| Consola de Redpanda | <http://localhost:8081> |
| Bandeja de Mailpit | <http://localhost:8025> |

!!! tip "El doble puerto de Redpanda no es capricho"
    Redpanda anuncia dos direcciones: `redpanda:9092` para los contenedores que viven en la misma red de Compose (como la consola), y `localhost:19092` para tu app corriendo **fuera** de Docker. Si usas el puerto equivocado, el cliente conecta al *bootstrap* y luego falla al hablar con el broker, porque le anunciaron un nombre que no puede resolver.

### Crear los topics

```bash
docker compose exec redpanda rpk topic create wallet.movements -p 1
docker compose exec redpanda rpk topic create wallet.movements-dlt -p 1
docker compose exec redpanda rpk topic list
```

!!! warning "El topic de la DLQ no se crea solo"
    `wallet.movements-dlt` es el *dead letter topic*: ahí van los eventos que no se pudieron procesar tras los reintentos (ver [Fase 8](fase-08-que-sigue.md)). Ojo con el nombre: Spring publica con sufijo **`-dlt`**, no `.DLT`. Créalo desde el principio, junto con el otro; si no existe, el consumidor del DLT arranca y muere.

### La configuración del perfil `local`

Spring carga `application-<perfil>.yaml` **encima** de `application.yaml` cuando ese perfil está activo. El `application.yaml` trae la config por defecto (la de la nube: Supabase, Redpanda con `SASL_SSL`, correo por Resend). Crea un `application-local.yaml` que reapunte todo a los contenedores:

```yaml title="src/main/resources/application-local.yaml"
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/wallet
    username: wallet
    password: wallet
  kafka:
    bootstrap-servers: localhost:19092
    properties:
      security.protocol: PLAINTEXT   # sin SASL/SSL en local
  mail:
    host: localhost
    port: 1025
    properties:
      mail.smtp.auth: false
      mail.smtp.starttls.enable: false

wallet:
  mail:
    from: wallet@baqjug.com
```

!!! abstract "Spring al paso: cómo gana el perfil"
    `application-local.yaml` solo lista lo que **cambia** en local, y al activar el perfil `local` esos valores **pisan** los de por defecto. Al poner `security.protocol: PLAINTEXT` y `bootstrap-servers: localhost:19092`, las credenciales SASL del archivo por defecto quedan **ignoradas** (bajo PLAINTEXT no aplican): no hace falta borrarlas. Y el correo se elige por perfil también (ver [Fase 10](fase-10-despliegue.md)): con `local` activo, el bean vivo es `SmtpMailPort` (`@Profile("local")`), que entrega a Mailpit; sin el perfil, sería `ResendMailPort`.

!!! danger "Si dejas `SASL_SSL` fijo, no conectas en local"
    El Redpanda de Compose corre sin SASL ni TLS; el de la nube exige ambos. Si el bloque `security.protocol: SASL_SSL` queda fijo en el `application.yaml` **sin** un `application-local.yaml` que lo pise, la app intentará autenticarse contra un broker local que no pide credenciales y **no conectará**. La seguridad va en la config por defecto (nube) y se apaga en el perfil `local`.

### Correr en local

```bash
SPRING_PROFILES_ACTIVE=local ./gradlew bootRun
```

En IntelliJ es lo mismo por la otra puerta: **Run → Edit Configurations → Environment variables**, agrega `SPRING_PROFILES_ACTIVE=local`, y corre `WalletApplication`. Con el perfil activo, la app toma el `application-local.yaml`: base, broker y correo apuntan a tus contenedores, sin exportar más variables.

### Verificar de punta a punta

Con el stack arriba y la app corriendo en perfil `local`, dispara una transferencia y comprueba cada eslabón de la cadena `transferencia → outbox → Kafka → movements → correo`:

```bash
curl -X POST http://localhost:8080/api/transfers \
  -H "Content-Type: application/json" \
  -d '{
        "fromId": "11111111-1111-1111-1111-111111111111",
        "toId": "22222222-2222-2222-2222-222222222222",
        "amount": 5000.00
      }'
```

Esperado, en orden:

1. La respuesta es `{"status":"COMPLETED"}`.
2. En los logs **no** aparece ningún `💀 Mensaje muerto` (nada cayó a la DLQ).
3. En Postgres, el movimiento quedó guardado y el evento marcado como procesado:

    ```sql
    SELECT from_id, to_id, amount FROM movements ORDER BY occurred_at DESC LIMIT 5;
    SELECT event_id, processed_at FROM processed_events ORDER BY processed_at DESC LIMIT 5;
    SELECT count(*) AS pendientes FROM outbox WHERE sent_at IS NULL;  -- baja a 0
    ```

4. En **<http://localhost:8025>** (Mailpit) está el correo "Recibiste un movimiento en tu wallet".

Conéctate a la base local con cualquier cliente en `localhost:5432` (base, usuario y clave `wallet`).

### Probar la DLQ en local

La [Fase 8](fase-08-que-sigue.md) explica el mecanismo; acá lo ejercitas contra tu Redpanda local. La idea: forzar que el consumidor **falle** con un evento envenenado para verlo aterrizar en `wallet.movements-dlt`.

Activa el centinela de demo (el `if` que lanza a propósito en el listener de `movement`, de la Fase 8) con una variable de entorno, para no dejarlo prendido por error:

```bash
SPRING_PROFILES_ACTIVE=local WALLET_DEMO_DLQ_SENTINEL=true ./gradlew bootRun
```

Produce un evento con monto negativo **directo al topic** (el endpoint no te deja mandar uno malo). El contenedor ya trae `rpk`:

```bash
echo '{"eventId":"00000000-0000-0000-0000-000000000000","fromId":"11111111-1111-1111-1111-111111111111","toId":"22222222-2222-2222-2222-222222222222","amount":-1,"occurredAt":"2026-01-01T00:00:00Z"}' \
  | docker compose exec -T redpanda \
    rpk topic produce wallet.movements --key 11111111-1111-1111-1111-111111111111
```

El listener lanza, el `DefaultErrorHandler` reintenta 3 veces y el evento aterriza en la DLQ. Míralo llegar:

```bash
docker compose exec redpanda rpk topic consume wallet.movements-dlt
```

Un mensaje malo no trancó la cola: se apartó a `wallet.movements-dlt` y los demás siguieron. Cuando termines, vuelve a arrancar **sin** `WALLET_DEMO_DLQ_SENTINEL`.

### Bajarlo

```bash
docker compose down -v    # el -v borra también los volúmenes (base limpia la próxima vez)
```

---

## Parte 2 — Redpanda Cloud (Serverless)

### Concepto al paso: qué es Redpanda

Redpanda es un broker compatible con el protocolo de Kafka: los mismos clientes, las mismas librerías, la misma configuración de Spring. Es más liviano y no necesita Zookeeper. **Serverless** es su modalidad gestionada: vive en la nube y se configura desde el navegador, sin Docker de por medio.

### Crear el clúster

1. Entra a [cloud.redpanda.com](https://cloud.redpanda.com) y crea tu cuenta (Google, GitHub o email). Redpanda genera tu *organization* automáticamente.
2. **Create cluster → Serverless**. Nombre (`wallet-workshop`), proveedor AWS, región `us-east-1`. Queda listo en segundos.
3. En **Overview → Kafka API**, copia el **bootstrap server**. Se ve así:

    ```
    d9taue875d5alho2qoe0.any.us-east-1.mpx.prd.cloud.redpanda.com:9092
    ```

!!! warning "No es una capa gratis permanente"
    Serverless es un *free trial* con créditos (alrededor de USD $100 por 14 días, sin tarjeta). Para un taller sobra, pero **borra el clúster al terminar**.

### Crear el usuario SASL

En **Security → Users → Create user**:

- Usuario: `workshop-wallet`
- Mecanismo: **SCRAM-SHA-256**
- Contraseña: la que genere

!!! danger "La contraseña se muestra una sola vez"
    Cópiala en el momento. Si la pierdes, toca borrar el usuario y crearlo de nuevo.

### Crear los topics

En **Topics → Create topic**, uno por uno, con 1 partición:

- `wallet.movements`
- `wallet.movements-dlt`

### Configurar las ACLs

Este es el paso donde se traba todo el mundo. Una ACL le dice al broker qué puede hacer un usuario sobre qué recurso. Sin ellas, el usuario existe, autentica bien y aun así no puede leer ni escribir nada.

En **Security → ACLs**, sobre el usuario `workshop-wallet`, necesitas **tres**:

| # | Resource Type | Pattern Type | Resource Name | Operation | Permission |
|---|---|---|---|---|---|
| 1 | Topic | `Prefixed` | `wallet.` | `All` | Allow |
| 2 | Consumer Group | `Any` | — | `All` | Allow |
| 3 | Cluster | — | — | `All` | Allow |

Host: `*` en las tres.

!!! warning "El punto del prefijo importa"
    Con `Literal` + `wallet`, la ACL solo cubre un topic llamado exactamente `wallet` — no cubre `wallet.movements`. Tiene que ser `Prefixed` + `wallet.`, con el punto.

!!! warning "Los consumer groups son un recurso aparte"
    La ACL del topic **no cubre** los grupos de consumidores. Si tus `group.id` son `movement`, `notification` y `dlt-demo` —ninguno con el prefijo `wallet.`—, una ACL de grupo con `Prefixed`/`wallet.` no les aplica y los consumidores no arrancan. Por eso la tabla usa `Any`: cubre cualquier `group.id` sin que tengas que enumerarlos. La alternativa prolija es renombrar los grupos (`wallet.notification`, `wallet.dlt-demo`) y usar el prefijo también aquí.

!!! note "Por qué la ACL de Cluster"
    El productor de Spring Boot trae `enable.idempotence=true` por defecto, y eso requiere el permiso `IdempotentWrite` a nivel de clúster. Sin él aparece un `ClusterAuthorizationException` que no menciona ningún topic y despista.

### Configuración en Spring Boot

```yaml title="application.yaml"
spring:
  kafka:
    bootstrap-servers: ${REDPANDA_BOOTSTRAP}
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: SCRAM-SHA-256
      sasl.jaas.config: >-
        org.apache.kafka.common.security.scram.ScramLoginModule required
        username="${REDPANDA_USER}"
        password="${REDPANDA_PASSWORD}";
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      properties:
        spring.json.trusted.packages: "com.baqjug.wallet.*"
    listener:
      # Sin esto, un fallo de permisos apaga el consumidor para siempre.
      auth-exception-retry-interval: 10s
```

!!! tip "`auth-exception-retry-interval` te salva el taller"
    Por defecto, Spring Kafka trata un fallo de autorización como fatal: registra `Fatal consumer exception; stopping container` y el consumidor **no vuelve a intentarlo**, ni siquiera cuando arreglas la ACL. Hay que reiniciar la app. Con este intervalo, reintenta cada 10 segundos y se levanta solo apenas el permiso aparece.

!!! note "El consumidor se endurece más en la Fase 8"
    Este bloque es lo mínimo para **conectar**. En la [Fase 8](fase-08-que-sigue.md) el consumidor suma dos cosas: los deserializadores envueltos en `ErrorHandlingDeserializer` (para que un JSON corrupto caiga a la DLQ y no quede en bucle) y `spring.json.use.type.headers: false` con un tipo por defecto (por el republicado del outbox). Acá no los repetimos.

### Verificar sin escribir código

```bash
rpk topic list \
  -X brokers=$REDPANDA_BOOTSTRAP \
  -X user=$REDPANDA_USER \
  -X pass=$REDPANDA_PASSWORD \
  -X sasl.mechanism=SCRAM-SHA-256 \
  -X tls.enabled=true
```

---

## Parte 3 — Integración continua con GitHub Actions

### Concepto al paso: CI/CD

**CI** (integración continua) es automatizar que, con cada `push`, el proyecto se compile y se corran las pruebas. **CD** (despliegue continuo) es que, si eso pasa, se despliegue solo. GitHub Actions ejecuta esos pasos en un *runner* —una máquina que GitHub te presta— siguiendo recetas guardadas en `.github/workflows/`.

### El workflow

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
          cache: gradle

      - name: Compilar y testear
        run: ./gradlew test
```

### Spring al paso: por qué los tests corren sin infraestructura

Los tests con MockK de la [Fase 4](fase-04-movimientos.md) no necesitan base ni broker: reemplazan las dependencias por dobles en memoria. Por eso el runner los ejecuta sin levantar nada. Los que sí requieren infraestructura real (con Testcontainers) también funcionan, porque el runner trae Docker — eso ya es un paso más avanzado.

!!! warning "El permiso de `gradlew`"
    En macOS el permiso de ejecución existe en tu disco, pero git puede no haberlo registrado. El runner corre Linux y falla con `permission denied: ./gradlew`. Verifícalo así:

    ```bash
    git ls-files -s gradlew    # debe empezar con 100755, no 100644
    git update-index --chmod=+x gradlew
    ```

### El reparto de responsabilidades

Actions hace el **CI** (compila y testea) y el host de despliegue hace el **CD** (construye la imagen y la publica; ver [Fase 10](fase-10-despliegue.md)). Así no hace falta guardar secretos de despliegue en Actions: cada herramienta se queda con lo suyo.

---

## Troubleshooting

Los errores reales que salen en este montaje, con su causa:

| Mensaje | Qué pasa realmente |
|---|---|
| `TOPIC_AUTHORIZATION_FAILED` | Puede ser falta de ACL **o que el topic no exista**. El broker responde igual en ambos casos para no revelar qué topics hay. Revisa primero que el topic exista con el nombre exacto (`wallet.movements-dlt`, con `-dlt`). |
| `GroupAuthorizationException: Not authorized to access group: X` | Falta la ACL de Consumer Group para ese `group.id`, o el grupo no cumple el prefijo de la ACL existente. |
| `ClusterAuthorizationException` | Falta la ACL de Cluster con `IdempotentWrite`. |
| `Fatal consumer exception; stopping container` | El consumidor se apagó y no reintenta. Configura `auth-exception-retry-interval` y reinicia la app. |
| `Bootstrap broker ... disconnected` en local | Puerto equivocado: desde el host es `localhost:19092`, no `9092`. |
| El cliente conecta y luego se cae | `--advertise-kafka-addr` mal configurado: el broker anuncia un nombre que tu app no resuelve. |
| `password authentication failed for user "postgres"` con Supabase | El *pooler* toma el usuario del propio JDBC URL. Agrega `?user=postgres.<ref-del-proyecto>` al final del URL. |
| `permission denied: ./gradlew` | Falta el bit de ejecución en git (`git update-index --chmod=+x gradlew`). |