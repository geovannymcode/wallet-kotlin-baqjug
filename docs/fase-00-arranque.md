# Fase 0 · El problema y el andamiaje

**Rama**: `fase-0`
**Lo que vas a lograr**: entender qué wallet vamos a construir y por qué, generar el proyecto con el asistente de Spring, dejarlo abierto en IntelliJ con la estructura por features, y conectarlo a Supabase.

---

## Parte 1 — El problema en una frase (5 min)

Una billetera digital tiene que hacer algo que suena trivial y no lo es: mover plata de una cuenta a otra sin que se pierda ni se duplique un peso. Si algo falla a la mitad, no puede quedar la cuenta de origen debitada y la de destino sin acreditar. Y cada movimiento tiene que quedar registrado.

Eso es lo que vamos a construir. Primero como un endpoint REST directo. Después, cuando ya funcione, le metemos eventos para desacoplar el registro y la notificación.

!!! abstract "Concepto al paso: qué es Spring Boot"
    Spring Boot es un framework para armar aplicaciones backend en la JVM sin configurar medio mundo a mano. Te da un servidor web embebido, conexión a base de datos, inyección de dependencias y un montón de piezas listas para usar. Tú te concentras en tu lógica; Boot cablea lo aburrido.

## Parte 2 — Generar el proyecto con Spring Initializr (10 min)

Spring Initializr es un asistente web que arma el esqueleto del proyecto. Entra a [https://start.spring.io](https://start.spring.io) y llena:

| Campo | Valor |
|-------|-------|
| Project | **Gradle - Kotlin** |
| Language | **Kotlin** |
| Spring Boot | **4.1.x** (o la 4.x estable más cercana) |
| Group | `com.baqjug` |
| Artifact | `wallet` |
| Package name | `com.baqjug.wallet` |
| Packaging | **Jar** |
| Java | **25** |

En **Dependencies**, agrega estas cuatro por ahora (la de Kafka la sumamos en la Fase 6):

| Dependencia | Para qué |
|-------------|----------|
| **Spring Web** | La API REST |
| **Spring Data JPA** | Guardar cuentas y movimientos |
| **PostgreSQL Driver** | El driver de Supabase |
| **Validation** | Validar los datos que entran |
| **Flyway Migration** | Versionar el esquema de la base |

Haz clic en **GENERATE**, descomprime dentro de tu repo y ábrelo en IntelliJ (**File → Open**). La primera vez baja el wrapper de Gradle y las dependencias. Espera a que termine.

!!! note "El plugin `kotlin-jpa` ya viene"
    Si generaste con JPA, el `build.gradle.kts` trae el plugin `kotlin("plugin.jpa")`. Ese plugin le genera por detrás a tus entidades el constructor sin argumentos que JPA exige, sin que tú tengas que escribirlo. Sin él, las entidades de Kotlin no arrancan.

## Parte 3 — La estructura por features (10 min)

Acá tomamos una decisión de organización. En vez de agrupar por capa técnica (un paquete `controllers`, otro `services`, otro `repositories`), agrupamos **por feature**: todo lo de cuentas junto, todo lo de transferencias junto, todo lo de movimientos junto.

Y dentro de cada feature separamos en dos: `api` e `internal`.

!!! abstract "Concepto al paso: `api` e `internal`"
    Piensa en una tienda. La `api` es la vitrina: lo que otros features pueden ver y usar (una interfaz, un DTO, un evento). El `internal` es la bodega: la entidad JPA, el repositorio, la implementación del servicio. Nadie de afuera entra a la bodega. Así, cuando alguien use la feature `account`, usa su vitrina y no se acopla a cómo está hecha por dentro.

Crea esta estructura dentro de `com.baqjug.wallet` (clic derecho → **New → Package**):

```
com.baqjug.wallet
├── account
│   ├── api          ← lo que ofrece la feature (interfaz, DTOs)
│   └── internal     ← entidad, repositorio, implementación
├── transfer
│   ├── api
│   └── internal
├── movement
│   ├── api
│   └── internal
└── notification     ← aparece en la Fase 7 (el consumidor)
```

No la llenamos toda ahora. Cada fase va poblando la feature que le toca.

## Parte 4 — Conectar Supabase (10 min)

Abre tu proyecto de Supabase, ve a **Project Settings → Database** y copia los datos de conexión. Vamos a pasárselos a Spring por variables de entorno, no quemados en el código.

Crea o edita `src/main/resources/application.properties`:

```properties title="src/main/resources/application.properties"
spring.application.name=wallet

# Supabase (Postgres). Los valores reales van por variables de entorno.
spring.datasource.url=${SUPABASE_DB_URL}
spring.datasource.username=${SUPABASE_DB_USER}
spring.datasource.password=${SUPABASE_DB_PASSWORD}

# JPA no crea tablas: de eso se encarga Flyway.
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.open-in-view=false

# Flyway busca las migraciones en db/migration.
spring.flyway.enabled=true
```

!!! abstract "Concepto al paso: variables de entorno"
    En vez de escribir la contraseña de la base dentro del código (donde terminaría en Git, a la vista de todos), la leemos de una variable de entorno con `${NOMBRE}`. En IntelliJ las configuras en **Run → Edit Configurations → Environment variables**. En producción las inyecta la plataforma. El código nunca sabe el secreto.

La URL de Supabase tiene esta forma (usa el host y puerto que te muestra tu panel):

```
jdbc:postgresql://TU-HOST.supabase.com:5432/postgres
```

!!! note "Por qué `ddl-auto=validate`"
    Le decimos a JPA que **no** cree ni modifique tablas. Solo que valide que lo que hay en la base concuerda con tus entidades. Quien manda sobre el esquema es Flyway, con migraciones versionadas. Dejar que Hibernate cree tablas solo está bien para un juguete; en algo que maneja plata, no.

## Cierre de la fase

Todavía no hay tablas ni entidades, así que la app aún no levanta contra la base. Eso es normal. Dejamos el andamiaje listo:

```bash
./gradlew compileKotlin
git add .
git commit -m "fase-0: proyecto wallet, estructura por features y conexión a Supabase"
git branch fase-0
```

En la [Fase 1](fase-01-dominio.md) modelamos la cuenta con su saldo, la primera migración de Flyway, y la app arranca por fin contra Supabase.
