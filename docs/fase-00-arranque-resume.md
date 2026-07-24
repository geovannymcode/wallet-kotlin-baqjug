# Fase 0 · El problema y el andamiaje - [MR]

**Rama**: `fase-0`
**Lo que vas a lograr**: 
- Entender el problema y sus requisitos, ver la arquitectura de un vistazo, elegir el stack justificándolo, generar el proyecto y organizarlo por features, y conectarlo a Supabase.

## Parte 1 — Planteamiento del problema - [MR]

Una billetera digital tiene que hacer algo que suena trivial y no lo es: 
- Mover dinero una cuenta a otra sin que se pierda ni se duplique un peso. Si algo falla a la mitad, no puede quedar la cuenta de origen debitada y la de destino sin acreditar. 
- Cada movimiento tiene que quedar registrado, y cuando ocurre, alguien va a querer enterarse (una alerta).

Lo vamos a construir primero como un endpoint REST directo y, cuando ya funcione, le adicionamos eventos para desacoplar el registro y la notificación.

## Parte 2 — Requerimiento | HU - [MR]
**HU**: Como usuario de la wallet, quiero consultar mi saldo y transferir plata a otra cuenta, con la certeza de que ni un centavo se pierde y de que cada movimiento queda registrado.

> 2.1 **Funcional — qué debe hacer:**
> Son las acciones que el sistema ofrece ("transferir plata").

- Consultar el saldo de una cuenta.
- Transferir plata de una cuenta a otra, rechazando la operación si no hay saldo suficiente.
- Registrar cada movimiento: origen, destino, monto, fecha y hora.
- Avisar cuando ocurre un movimiento (notificación). Esto llega en la parte de eventos.

> 2.2 **No funcional — cómo debe comportarse:**
> Son cualidades de cómo las hace ("nunca dejar un saldo roto", "responder rápido").

- **Consistencia**: una transferencia es todo-o-nada. Jamás puede quedar la cuenta de origen debitada y la de destino sin acreditar.
- **Exactitud del dinero**: ni un centavo perdido por redondeo.
- **Evolución sin riesgo**: poder agregar reacciones nuevas (correo, antifraude, métricas) sin tocar el corazón de la transferencia.
- **Trazabilidad**: cada cuenta y cada movimiento llevan registro de cuándo ocurrieron.


> 2.3 **Restricciones — los límites duros:**
> Son reglas duras que acotan la solución ("el dinero se representa exacto").

- El dinero se representa con precisión exacta; nada de números flotantes.
- No se puede perder un movimiento ni procesarlo mal porque un proceso falló.
- El esquema de la base viaja versionado en el repositorio, reproducible en cualquier máquina.

## Parte 3 — Arquitectura y Diseño - [MR]
Diagrama de la solución.

![Arquitectura general: los clientes (consumers) hacen peticiones REST a la aplicación wallet, que a través de ORM/JPA lee y escribe las tablas de cuentas y transferencias en la base de datos Postgres](img/Img_0.png)
Esa forma nos alcanza para las primeras fases. Más adelante, cuando entren los eventos, a la derecha aparecerá un broker por donde salen los hechos que otros servicios consumen; pero el corazón sigue siendo el mismo: clientes → aplicación → base.

## Parte 4 — Stack Tecnológico - [MR]
Recién ahora elegimos herramientas, y cada una responde a un requisito de arriba, no a la moda:

| Nombre     | Descripción                              |
|------------|------------------------------------------|
| Dependencies | **Gradle**                      |
| Language   | **Kotlin**                               |
| Spring Boot | **4.1.x** (o la 4.x estable más cercana) |
| Group      | `com.baqjug`                             |
| Artifact   | `wallet`                                 |
| Package name | `com.baqjug.wallet`                      |
| Packaging  | **Jar**                                  |
| Java       | **25**                                   |
| Postgres   | Database - Supabase                      |
| Restful    | API REST - JSON format                   |

En **Dependencies**, agrega estas por ahora (la de Kafka la sumamos en la Fase 6):

| Dependencia | Para qué                        |
|-------------|---------------------------------|
| **Spring Web** | La API REST                     |
| **Spring Data JPA** | Guardar cuentas y movimientos   |
| **PostgreSQL Driver** | El driver de conexión           |
| **Validation** | Validar los datos que entran    |
| **Flyway Migration** | Versionar el esquema de la base |

## Parte 5 — Estructura del Proyecto  - [GEO]
Crea esta estructura dentro de `com.baqjug.wallet`

```
com.baqjug.wallet
├── account
│   ├── domain      ← entidad, repositorio, servicio, mapper, DTOs, excepciones
│   └── web         ← el controlador REST
├── transfer
│   ├── domain
│   └── web
├── movement
│   ├── domain      ← entidad, repositorio, servicio
│   └── messaging   ← aparece con los eventos (Fases 6-7): consume del broker
├── notification    ← aparece en la Fase 7 (consumidor por eventos)
│   └── messaging
└── web
    └── exception   ← manejo de errores compartido (GlobalExceptionHandler)
```
### Dentro de cada feature: `domain` y `web`
Cada feature se separa en dos sub-carpetas:

- **`domain`** es el corazón. Acá vive la lógica: la entidad que se guarda en la base, el repositorio, el servicio con las reglas de negocio, el mapper y los objetos de datos. El domain **no sabe nada de HTTP** ni de quién lo llama. Podrías invocarlo desde un método "main()", desde una tarea programada o desde un test, y le daría exactamente igual.
- **`web`** es la puerta de entrada por HTTP: el controlador. Su único trabajo es recibir la petición web, sacar los datos y pasárselos al `domain`. Cero lógica de negocio acá.

!!! abstract "Concepto al paso: 
- ¿por qué separar `domain` de `web`?"
    - Porque el "cómo entra" (HTTP hoy, una cola de mensajes mañana, la línea de comandos pasado) no debería ensuciar el "qué hace". 
    - Si la lógica vive en `domain` sin depender de la web, el día que además la quieras disparar por un evento —cosa que haremos en la Fase 7— no reescribes nada: la vuelves a llamar desde otra puerta. El `domain` es reusable; la puerta es intercambiable.

## Parte 6 — Conectar a BD Postgres (Supabase) - [MR]

- Abre tu proyecto de Supabase, ve a **Project Settings → Database** y copia los datos de conexión. 
- Vamos a pasárselos a Spring por variables de entorno, no quemados en el código.
- Pasos: Opción Connect - Tipo "Direct" - Direct Connect - URI - Info connection.
- Crea o edita `src/main/resources/application.yaml`:

```yaml title="src/main/resources/application.yaml"
spring:
  application:
    name: wallet

  # Supabase (Postgres). Los valores reales van por variables de entorno.
  datasource:
    url: ${SUPABASE_DB_URL}
    username: ${SUPABASE_DB_USER}
    password: ${SUPABASE_DB_PASSWORD}

  # JPA no crea tablas: de eso se encarga Flyway.
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false

  # Flyway busca las migraciones en db/migration.
  flyway:
    enabled: true
```

!!! abstract "Concepto al paso: variables de entorno"
- En vez de escribir la contraseña de la base dentro del código (donde terminaría en Git, a la vista de todos), la leemos de una variable de entorno con `${NOMBRE}`. 
- En IntelliJ las configuras en **Run → Edit Configurations → Environment variables**. En producción las inyecta la plataforma. El código nunca sabe el secreto.

La URL de BD tiene esta forma (usa el host y puerto que te muestra tu panel):

```
postgresql://postgres:[PASSWORD]@db.{{HOST}}.supabase.co:5432/postgres
jdbc:postgresql://TU-HOST.supabase.com:5432/postgres
```

!!! note "Por qué `ddl-auto=validate`"
- Le decimos a JPA que **no** cree ni modifique tablas. Solo que valide que lo que hay en la base concuerda con tus entidades. Quien manda sobre el esquema es Flyway, con migraciones versionadas. 
- Dejar que Hibernate cree tablas solo está bien para un juguete; en algo que maneja plata, no.

## Paso 7. Cierre de la fase - [GEO]

Todavía no hay tablas ni entidades, así que la app aún no levanta contra la base. Eso es normal. Dejamos el andamiaje listo:

```bash
./gradlew compileKotlin
git add .
git commit -m "fase-0: proyecto wallet, estructura por features y conexión a Supabase"
git branch fase-0
```

En la [Fase 1](fase-01-dominio.md) modelamos la cuenta con su saldo, la primera migración de Flyway, y la app arranca por fin contra Supabase.
