# Referencias

Para seguir por tu cuenta, sin relleno. Estas son las fuentes que valen para lo que tocamos.

## Kotlin

- [Documentación oficial de Kotlin](https://kotlinlang.org/docs/home.html). Empieza por data classes, sealed classes y null safety, que es lo que más usamos.
- [Kotlin para desarrolladores de Java](https://kotlinlang.org/docs/comparison-to-java.html). Si vienes de Java, esto te ahorra confusiones.
- [Kotlin Coroutines](https://kotlinlang.org/docs/coroutines-overview.html). Funciones suspendidas, `coroutineScope`, `async`/`await` y concurrencia estructurada, para la Fase 9.

## Spring Boot y Spring Data

- [Spring Boot Reference](https://docs.spring.io/spring-boot/index.html). La sección de datos y la de Kafka.
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/index.html). Para entender qué te regala `JpaRepository` y cómo escribir consultas derivadas.
- [Guía de Spring sobre `@Transactional`](https://docs.spring.io/spring-framework/reference/data-access/transaction.html). Vale leerla completa; casi nadie lo hace.

## Flyway

- [Documentación de Flyway](https://documentation.red-gate.com/flyway). La convención de nombres y por qué no se editan migraciones ya aplicadas.

## Eventos, Kafka y Redpanda

- [Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/index.html). Productores, consumidores, serialización JSON, reintentos y DLQ.
- [Documentación de Redpanda](https://docs.redpanda.com). El setup de Serverless y el uso de `rpk`.
- Para el patrón outbox y la idempotencia, busca "Transactional Outbox Pattern" de Chris Richardson (microservices.io). Es la explicación de referencia.

## Despliegue y correo

- [Documentación de GitHub Actions](https://docs.github.com/actions). Workflows, runners, secretos y el marketplace de acciones.
- [Documentación de Render](https://render.com/docs) y [de Railway](https://docs.railway.com). Desplegar desde un `Dockerfile`, variables de entorno y deploy hooks.
- [Mailpit](https://mailpit.axllent.org/docs/). El servidor SMTP falso con UI web para atrapar correos en local.
- [Spring Boot: sending email](https://docs.spring.io/spring-boot/reference/io/email.html). `JavaMailSender` y la autoconfiguración de `spring.mail.*`.

## Supabase

- [Documentación de Supabase](https://supabase.com/docs). Nos interesa la parte de conexión directa a Postgres y el manejo de credenciales.
