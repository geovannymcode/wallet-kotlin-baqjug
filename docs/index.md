# Una wallet en Kotlin, de un endpoint REST a eventos

Esta guía construye desde cero una billetera digital en Kotlin y Spring Boot. Empezamos con un endpoint REST que mueve plata entre dos cuentas contra una base Postgres, y terminamos con ese mismo movimiento viajando como un evento que otro servicio consume. Se puede seguir sola, de principio a fin.

Si nunca tocaste Kotlin, tranquilo: vamos explicando cada cosa del lenguaje la primera vez que aparece. Si vienes de Kotlin pero nunca hiciste Spring, igual: cada anotación y cada pieza de infraestructura se explica cuando entra en juego. No asumo que porque lees una cosa ya sabes la otra.

!!! note "Esto no es un 'hola mundo'"
    Aunque explicamos todo desde la base, lo que construimos tiene sustancia: dinero con `BigDecimal`, una transferencia transaccional que no puede dejar saldos rotos, y un desacople real por mensajería. Los conceptos difíciles no se esconden, se explican.

## Qué vamos a construir

Una wallet, **`wallet`**, con tres cosas que hace cualquier billetera:

- Consultar el saldo de una cuenta.
- Transferir plata de una cuenta a otra, validando que haya saldo.
- Dejar registrado cada movimiento.

A vista de pájaro, la wallet es esto: unos clientes entran por HTTP a una aplicación REST, y esa aplicación lee y escribe en una base de datos Postgres. Nada más. Toda la magia de las siguientes fases pasa dentro de esa caja del medio.

![Arquitectura general: los clientes (consumers) hacen peticiones REST a la aplicación wallet, que a través de ORM/JPA lee y escribe las tablas de cuentas y transferencias en la base de datos Postgres](img/Img_0.png)

Esa caja del medio, la aplicación, es donde vive todo lo que vamos a escribir. Y adentro, el registro de cada movimiento arranca de la forma más simple posible: una llamada directa dentro del mismo proceso.

![Flujo síncrono: la petición REST entra a transfer, que valida el saldo y debita/acredita en account, y registra el movimiento en movement con una llamada directa dentro del mismo proceso](img/Img_1.png)

En la parte de eventos, el registro deja de ser una llamada directa y pasa a publicarse:

![Flujo por eventos: transfer publica el evento MovimientoRegistrado en el topic de Redpanda, y dos grupos de consumidores lo reciben cada uno por su cuenta: notification (envía el correo) y movement (guarda el registro)](img/Img_2.png)

El cambio se ve chico en el diagrama, pero es el corazón de la segunda parte: `transfer` deja de saber quién registra o notifica. Solo suelta un evento. Quién lo escuche, y cuántos lo escuchen, ya no es su problema.

## El problema que motiva la parte de eventos

Arranca simple: una transferencia mueve plata y registra el movimiento. Pero el negocio nunca se queda quieto. A los tres meses, cada vez que ocurre una transferencia te piden algo más: manda un correo al que recibió, dispara un push, avísale a antifraude, suma puntos de fidelidad, refresca el tablero de métricas. Seis cosas, y contando.

Si todas cuelgan de `transfer` con una llamada directa, pasan tres cosas, y las tres son malas:

- `transfer` se vuelve un **monstruo que conoce a media empresa**: importa el servicio de correo, el de push, el de antifraude, y cada feature nueva es editar este método y volver a desplegarlo, aunque la lógica de mover plata no cambió en nada.
- Se **amarra el destino de todos**: si el servicio de correo está lento, la transferencia se pone lenta; si se cae, la transferencia se cae **con él**. El cliente no puede mover su plata porque un correo no salió. Absurdo, pero es lo que pasa cuando todo va en la misma llamada.
- El que llega tarde **arriesga lo que ya servía**: meter antifraude te obliga a tocar —y poner en riesgo— el código que ya movía plata bien.

La mensajería por eventos corta ese nudo de raíz. `transfer` hace lo suyo, mover la plata, y publica un hecho: *"se registró un movimiento"*. Ahí termina su responsabilidad. Quién quiera enterarse se suscribe y reacciona por su cuenta —el correo, el push, antifraude—, cada uno en su proceso, cayéndose y levantándose sin arrastrar a nadie. Y lo mejor: agregar un consumidor nuevo **no toca `transfer`**: ni lo editas, ni lo redespliegas, ni lo arriesgas.

Eso es lo que vas a construir con tus manos de la Fase 5 en adelante: pasar de una llamada directa y frágil a un hecho publicado que desacopla toda la cadena.

## Cómo está organizada

Cada fase es una etapa y, en el repo, una rama. Puedes hacer `git checkout` a cualquiera para seguir desde ahí si te atrasas.

| Fase | Rama | Qué construye |
|------|------|----------------|
| 0 | `fase-0` | El problema, el andamiaje y Supabase |
| 1 | `fase-1` | El dominio: cuenta, saldo, JPA y Flyway |
| 2 | `fase-2` | Exponer el saldo por REST |
| 3 | `fase-3` | Transferencia con validación de saldo |
| 4 | `fase-4` | Registro de movimientos (llamada directa) |
| 5 | `fase-5` | Por qué eventos: broker, topic, producer, consumer |
| 6 | `fase-6` | Publicar el evento con Redpanda |
| 7 | `fase-7` = `main` | Consumir el evento desde otro servicio |
| 8 | (referencia) | De demo a producción: outbox, idempotencia, DLQ |
| 9 | (opcional) | Coroutines: un consumidor no bloqueante |
| 10 | (despliegue) | Docker, GitHub Actions y deploy a la nube |

!!! tip "Puedes entrar por cualquier fase"
    Cada rama deja el proyecto justo en el estado de esa fase. Si te perdiste en un paso, `git checkout fase-2` y sigues desde ahí sin quedarte atrás. Y si lo que te interesa es directamente la parte de eventos, `git checkout fase-4` te pone en la línea de salida con toda la base REST ya lista. La rama `main` es la versión final con eventos.

!!! note "El corazón y las extensiones"
    Las Fases 0 a 7 son el hilo principal: de un endpoint REST a eventos, con una wallet completa al final. La 8 es referencia (lo que falta para producción: outbox, idempotencia, DLQ), la 9 es un extra avanzado y opcional (coroutines), y la 10 lleva la app a la nube (Docker, integración continua y despliegue). Puedes parar en la 7 y ya tienes algo redondo, o seguir hasta donde quieras.

## Versiones

Kotlin 2.3 · Spring Boot 4.1 · JDK 25 · Spring Data JPA · Flyway · Spring for Apache Kafka · Supabase (PostgreSQL 17) · Redpanda · Docker · GitHub Actions.

!!! note "Confirma versiones en el asistente"
    En [start.spring.io](https://start.spring.io) las versiones cambian con el tiempo. Si no ves Spring Boot 4.1, elige la 4.x estable más cercana. El asistente se encarga de que las dependencias sean compatibles entre sí.

Cuando estés listo, revisa [Requisitos y setup](requisitos.md) y arranca por la [Fase 0](fase-00-arranque.md).
