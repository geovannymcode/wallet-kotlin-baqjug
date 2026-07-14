# Una wallet en Kotlin, de un endpoint REST a eventos

Esta guía construye desde cero una billetera digital en Kotlin y Spring Boot. Empezamos con un endpoint REST que mueve plata entre dos cuentas contra una base Postgres, y terminamos con ese mismo movimiento viajando como un evento que otro servicio consume. Se puede seguir sola, de principio a fin.

El material sirve para dos talleres distintos con **un solo proyecto**:

- En **BAQJUG / IDITEK** cubrimos de la Fase 0 a la Fase 4: la wallet funcionando por REST, con transferencias, validación de saldo y registro de movimientos contra Supabase.
- En **CaribeConf** arrancamos también desde cero, pero el foco es la última parte: convertir ese registro de movimientos en un evento y consumirlo desde otro servicio con Redpanda.

Si nunca tocaste Kotlin, tranquilo: vamos explicando cada cosa del lenguaje la primera vez que aparece. Si vienes de Kotlin pero nunca hiciste Spring, igual: cada anotación y cada pieza de infraestructura se explica cuando entra en juego. No asumo que porque lees una cosa ya sabes la otra.

!!! note "Esto no es un 'hola mundo'"
    Aunque explicamos todo desde la base, lo que construimos tiene sustancia: dinero con `BigDecimal`, una transferencia transaccional que no puede dejar saldos rotos, y un desacople real por mensajería. Los conceptos difíciles no se esconden, se explican.

## Qué vamos a construir

Una wallet, **`wallet`**, con tres cosas que hace cualquier billetera:

- Consultar el saldo de una cuenta.
- Transferir plata de una cuenta a otra, validando que haya saldo.
- Dejar registrado cada movimiento.

Al principio, ese registro es una llamada directa dentro del mismo proceso:

```
[ REST ] → transfer → account (valida saldo, debita/acredita)
                    → movement (registra el movimiento)   ← llamada directa
```

En la parte de eventos, el registro deja de ser una llamada directa y pasa a publicarse:

```
[ REST ] → transfer → account
                    → publica "MovimientoRegistrado"  →  [ Redpanda ]
                                                              │
                                        notification (consume y "notifica")
```

El cambio se ve chico en el diagrama, pero es el corazón de la segunda parte: `transfer` deja de saber quién registra o notifica. Solo suelta un evento. Quién lo escuche, y cuántos lo escuchen, ya no es su problema.

## El problema que motiva la parte de eventos

Imagina que a la wallet le empiezan a pedir cosas cada vez que ocurre una transferencia: registra el movimiento, manda un correo, dispara una notificación push, avisa a antifraude, actualiza un tablero. Si todo eso lo llama `transfer` de forma directa, ese método se vuelve un monstruo que conoce a media empresa. Y si el servicio de correo se cae, la transferencia se cae con él.

La mensajería por eventos corta ese nudo. `transfer` publica un hecho, "esto pasó", y sigue con su vida. Los interesados escuchan por su cuenta. Se caen y se levantan sin arrastrar a la transferencia. Eso es lo que vas a construir con tus manos en la Fase 6 y 7.

## Cómo está organizada

Cada fase es una etapa y, en el repo, una rama. Puedes hacer `git checkout` a cualquiera para seguir desde ahí si te atrasas.

| Fase | Rama | Qué construye | Evento |
|------|------|----------------|--------|
| 0 | `fase-0` | El problema, el andamiaje y Supabase | IDITEK |
| 1 | `fase-1` | El dominio: cuenta, saldo, JPA y Flyway | IDITEK |
| 2 | `fase-2` | Exponer el saldo por REST | IDITEK |
| 3 | `fase-3` | Transferencia con validación de saldo | IDITEK |
| 4 | `fase-4` | Registro de movimientos (llamada directa) | IDITEK |
| 5 | `fase-5` | Por qué eventos: broker, topic, producer, consumer | CaribeConf |
| 6 | `fase-6` | Publicar el evento con Redpanda | CaribeConf |
| 7 | `fase-7` = `main` | Consumir el evento desde otro servicio | CaribeConf |
| 8 | (referencia) | Qué sigue: outbox, idempotencia, DLQ | CaribeConf |

!!! tip "Si solo vienes a CaribeConf"
    Arrancas desde cero igual. Las Fases 0 a 4 quedan documentadas acá como referencia: puedes leerlas antes o hacer `git checkout fase-4` para ponerte en la línea de salida de la parte de eventos. En el taller en vivo pasamos rápido por esa base y le dedicamos el tiempo a lo nuevo, publicar y consumir eventos.

!!! tip "Para seguir a tu ritmo en cualquiera de los dos"
    Cada rama deja el proyecto justo en el estado de esa fase. Si te perdiste en un paso, `git checkout fase-2` y sigues desde ahí sin quedarte atrás. La rama `main` es la versión final con eventos.

## Versiones

Kotlin 2.3 · Spring Boot 4.1 · JDK 25 · Spring Data JPA · Flyway · Spring for Apache Kafka · Supabase (PostgreSQL 17) · Redpanda.

!!! note "Confirma versiones en el asistente"
    En [start.spring.io](https://start.spring.io) las versiones cambian con el tiempo. Si no ves Spring Boot 4.1, elige la 4.x estable más cercana. El asistente se encarga de que las dependencias sean compatibles entre sí.

Cuando estés listo, revisa [Requisitos y setup](requisitos.md) y arranca por la [Fase 0](fase-00-arranque.md).
