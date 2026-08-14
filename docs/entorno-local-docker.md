# Entorno local con Docker

A lo largo del taller usaste servicios en la nube (Supabase para Postgres, Redpanda Cloud para eventos) y Mailpit local para el correo. Eso está perfecto para aprender. Pero la app está hecha para correr **igual de bien 100% en tu máquina**: la misma imagen, la misma lógica, cambiando solo la configuración por **perfil**. Acá levantas todo el stack —Postgres, Redpanda y Mailpit— con un solo `docker compose`, sin depender de ninguna cuenta en la nube.

!!! note "Nube vs local: qué cambia y qué no"
    El **código de negocio no cambia**. Lo único que cambia es la configuración: con el perfil `local`, la app apunta a los contenedores de tu máquina y manda el correo por SMTP a Mailpit; sin ese perfil (nube), apunta a Supabase/Redpanda Cloud y manda el correo por la API de Resend (ver [Fase 10](fase-10-despliegue.md)). Un mismo binario, dos entornos.

---

## Parte 1 — El stack local con `docker compose`

Tres servicios: la base, el broker y el buzón de correo falso. Deja este archivo en la raíz del proyecto:

```yaml title="docker-compose.yml"
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: wallet
      POSTGRES_USER: wallet
      POSTGRES_PASSWORD: wallet
    ports:
      - "5432:5432"

  redpanda:
    image: docker.redpanda.com/redpandadata/redpanda:latest
    command:
      - redpanda
      - start
      - --smp=1
      - --overprovisioned
      - --kafka-addr=PLAINTEXT://0.0.0.0:9092
      - --advertise-kafka-addr=PLAINTEXT://localhost:9092
    ports:
      - "9092:9092"

  mailpit:
    image: axllent/mailpit
    ports:
      - "1025:1025"   # SMTP (por acá entrega la app)
      - "8025:8025"   # UI web (acá ves los correos)
```

Levántalo:

```bash
docker compose up -d
```

- **Postgres** queda en `localhost:5432` (base `wallet`, usuario/clave `wallet`). Flyway corre las migraciones `V1`–`V4` al arrancar la app, así que las tablas (`accounts`, `movements`, `outbox`, `processed_events`) y las cuentas semilla se crean solas.
- **Redpanda** queda en `localhost:9092`, hablando el protocolo de Kafka **en texto plano** (sin SASL/SSL): en local no necesitas autenticarte. Y a diferencia de la nube, **auto-crea los topics**, así que `wallet.movements` y `wallet.movements-dlt` aparecen solos cuando la app los usa.
- **Mailpit** atrapa el correo en `localhost:1025` y te lo muestra en `http://localhost:8025`.

!!! note "Opcional: una UI para ver el broker"
    Si quieres ver los topics y mensajes en el navegador (útil para la demo de DLQ), suma **Redpanda Console** al `docker-compose.yml` (`docker.redpanda.com/redpandadata/console`) apuntando a `redpanda:9092`, o usa AKHQ/Kafdrop. No es necesario para que la app corra.

## Parte 2 — La configuración del perfil `local`

Spring carga `application-<perfil>.yaml` **encima** de `application.yaml` cuando ese perfil está activo. Crea un `application-local.yaml` que reapunte todo a los contenedores:

```yaml title="src/main/resources/application-local.yaml"
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/wallet
    username: wallet
    password: wallet
  kafka:
    bootstrap-servers: localhost:9092
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
    `application.yaml` trae la config **por defecto** (la de la nube: Supabase, Redpanda con `SASL_SSL`, Resend). `application-local.yaml` solo lista lo que **cambia** en local, y al activar el perfil `local` esos valores **pisan** los de por defecto. Al poner `security.protocol: PLAINTEXT` y `bootstrap-servers: localhost:9092`, las credenciales SASL del archivo por defecto quedan **ignoradas** (bajo PLAINTEXT no aplican). No hace falta borrarlas.

Recuerda que el correo también se elige por perfil (ver [Fase 10](fase-10-despliegue.md)): con `local` activo, el bean vivo es `SmtpMailPort` (`@Profile("local")`), que entrega a Mailpit. Sin el perfil, sería `ResendMailPort`.

## Parte 3 — Correr la app en local

Activa el perfil `local` y arranca:

```bash
SPRING_PROFILES_ACTIVE=local ./gradlew bootRun
```

En IntelliJ es lo mismo por la otra puerta: **Run → Edit Configurations → Environment variables**, agrega `SPRING_PROFILES_ACTIVE=local`, y corre `WalletApplication`.

## Parte 4 — Verificar de punta a punta

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

4. En **`http://localhost:8025`** (Mailpit) está el correo "Recibiste un movimiento en tu wallet".

Conéctate a la base local con cualquier cliente en `localhost:5432` (base `wallet`, usuario/clave `wallet`).

## Parte 5 — Probar la DLQ en local

La [Fase 8](fase-08-que-sigue.md) explica el mecanismo; acá lo ejercitas contra tu Redpanda local. La idea: forzar que el consumidor **falle** con un evento envenenado para verlo aterrizar en `wallet.movements-dlt`.

Activa el centinela de demo (el `if` que lanza a propósito en el listener de `movement`, de la Fase 8) con una variable de entorno, para no dejarlo prendido por error:

```bash
SPRING_PROFILES_ACTIVE=local WALLET_DEMO_DLQ_SENTINEL=true ./gradlew bootRun
```

Produce un evento con monto negativo **directo al topic** (el endpoint no te deja mandar uno malo). El contenedor de Redpanda ya trae `rpk`:

```bash
echo '{"eventId":"00000000-0000-0000-0000-000000000000","fromId":"11111111-1111-1111-1111-111111111111","toId":"22222222-2222-2222-2222-222222222222","amount":-1,"occurredAt":"2026-01-01T00:00:00Z"}' \
  | docker exec -i $(docker compose ps -q redpanda) \
    rpk topic produce wallet.movements --key 11111111-1111-1111-1111-111111111111
```

El listener lanza, el `DefaultErrorHandler` reintenta 3 veces y el evento aterriza en la DLQ. Míralo llegar:

```bash
docker exec -it $(docker compose ps -q redpanda) rpk topic consume wallet.movements-dlt
```

Un mensaje malo no trancó la cola: se apartó a `wallet.movements-dlt` y los demás siguieron. Cuando termines la demo, vuelve a arrancar **sin** `WALLET_DEMO_DLQ_SENTINEL`.

## Parte 6 — Apagar el stack

```bash
docker compose down          # detiene los contenedores
docker compose down -v       # además borra los datos (base limpia la próxima vez)
```

Con esto tienes la wallet corriendo entera en tu máquina, sin cuentas en la nube: ideal para desarrollar, probar la cadena completa de eventos, o preparar la demo de DLQ. Para llevarla a internet, sigue en la [Fase 10](fase-10-despliegue.md).
