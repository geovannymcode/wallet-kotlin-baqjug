# Punto de partida · Si arrancas por los eventos

Este taller se puede seguir completo, de la Fase 0 a la 10. Pero si llegas para la **parte de eventos** —de la Fase 5 en adelante—, esta página es tu punto de partida: te pone en contexto y te deja el proyecto listo para arrancar, sin construir toda la base en vivo.

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
