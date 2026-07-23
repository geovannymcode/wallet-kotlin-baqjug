# Fase 0 · El problema y el andamiaje

**Rama**: `fase-0`
**Lo que vas a lograr**: entender qué wallet vamos a construir y por qué, generar el proyecto con el asistente de Spring, dejarlo abierto en IntelliJ con la estructura por features, y conectarlo a Supabase.

---

## Parte 1 — El problema en una frase

Una billetera digital tiene que hacer algo que suena trivial y no lo es: mover plata de una cuenta a otra sin que se pierda ni se duplique un peso. Si algo falla a la mitad, no puede quedar la cuenta de origen debitada y la de destino sin acreditar. Y cada movimiento tiene que quedar registrado.

Eso es lo que vamos a construir. Primero como un endpoint REST directo. Después, cuando ya funcione, le metemos eventos para desacoplar el registro y la notificación.

!!! abstract "Concepto al paso: qué es Spring Boot"
    Spring Boot es un framework para armar aplicaciones backend en la JVM sin configurar medio mundo a mano. Te da un servidor web embebido, conexión a base de datos, inyección de dependencias y un montón de piezas listas para usar. Tú te concentras en tu lógica; Boot cablea lo aburrido.

## Parte 2 — Generar el proyecto con Spring Initializr

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

## Parte 3 — Cómo organizamos el código

Acá tomamos la decisión de organización más importante del taller, y la vamos a explicar con calma porque de esto depende cómo se ve todo lo que sigue. Si nunca has pensado en "arquitectura de software", tranquilo: por ahora es solo **dónde poner cada archivo y por qué**.

### La forma común (y por qué no la usamos)

Cuando arrancas, lo típico es agrupar el código por **capa técnica**: una carpeta `controllers` con todos los controladores, otra `services` con todos los servicios, otra `repositories` con todos los repositorios. Funciona para algo chiquito. Pero a medida que crece, para tocar "lo de transferencias" terminas saltando entre tres o cuatro carpetas lejanas, porque todo lo de una misma cosa quedó regado.

### Lo que sí hacemos: agrupar por feature

Nosotros agrupamos **por feature**: todo lo de cuentas junto, todo lo de transferencias junto, todo lo de movimientos junto. Abres la carpeta `account` y ahí está TODO lo de cuentas: la entidad, el repositorio, el servicio, el controlador. No tienes que buscar en cinco lados.

!!! abstract "Concepto al paso: ¿qué es una 'feature'?"
    Una feature es una capacidad del sistema con sentido de negocio: "cuentas", "transferencias", "movimientos". No es una capa técnica ("controladores"), es algo que el negocio reconocería. La regla que seguimos viene de un enfoque llamado **Tomato Architecture**, que en un monolito dice: **primero divide por feature**, y solo adentro de cada feature separa por capas.

Esto no es un capricho de estilo. Es el principio #1 de Tomato Architecture, un enfoque pragmático cuyo lema es "no te compliques": nada de puertos, adaptadores ni abstracciones "por si algún día cambiamos de base de datos". Código simple y aburrido, que es el que sobrevive años y el que un compañero nuevo entiende sin sufrir.

### Dentro de cada feature: `domain` y `web`

Cada feature se parte en dos sub-carpetas:

- **`domain`** es el corazón. Acá vive la lógica: la entidad que se guarda en la base, el repositorio, el servicio con las reglas de negocio, el mapper y los objetos de datos. El `domain` **no sabe nada de HTTP** ni de quién lo llama. Podrías invocarlo desde un `main()`, desde una tarea programada o desde un test, y le daría exactamente igual.
- **`web`** es la puerta de entrada por HTTP: el controlador. Su único trabajo es recibir la petición web, sacar los datos y pasárselos al `domain`. Cero lógica de negocio acá.

!!! abstract "Concepto al paso: ¿por qué separar `domain` de `web`?"
    Porque el "cómo entra" (HTTP hoy, una cola de mensajes mañana, la línea de comandos pasado) no debería ensuciar el "qué hace". Si la lógica vive en `domain` sin depender de la web, el día que además la quieras disparar por un evento —cosa que haremos en la Fase 7— no reescribes nada: la vuelves a llamar desde otra puerta. El `domain` es reusable; la puerta es intercambiable.

### Dos cosas que NO vamos a hacer (y por qué)

Estas dos decisiones van a contramano de mucho tutorial que verás por ahí, así que las explico bien:

**1. No creamos interfaces "por si acaso".** En muchos proyectos, por cada `AccountService` hay una interfaz `AccountService` y una clase `AccountServiceImpl` que la implementa, aunque solo exista UNA implementación. Eso es peso muerto: dos archivos para una sola cosa. Nuestro `AccountService` va a ser una **clase concreta y ya**. ¿Que mañana necesitas una segunda implementación? El IDE te extrae la interfaz en dos teclas. ¿Que la necesitas para un test? Las librerías de mocking mockean clases sin problema. Crear la interfaz desde el día uno es resolver un problema que casi nunca llega.

!!! note "La única excepción: los repositorios"
    Vas a ver que `AccountRepository` sí es una interfaz. Pero no es idea nuestra: Spring Data **exige** una interfaz para generarte la implementación por detrás (lo vemos en la Fase 1). Es una interfaz que el framework nos obliga a tener, no una que inventamos "por si acaso". Esa es toda la diferencia.

**2. Separamos la entidad de la base de lo que sale por la API.** La clase que se guarda en Postgres (la llamaremos `AccountEntity`) NO es la misma que devolvemos en el JSON (`AccountResponse`). Un pequeño **mapper** traduce de una a la otra. Suena a trabajo extra, pero te salva de que un cambio en la tabla te cambie sin querer el contrato de tu API. Lo ves en acción en la Fase 2.

### La estructura, en un diagrama

Crea esta estructura dentro de `com.baqjug.wallet` (clic derecho → **New → Package**). No la llenamos toda ahora; cada fase puebla lo que le toca:

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

!!! note "¿De dónde sale todo esto?"
    Este patrón lo popularizó Siva Prasad Reddy con el nombre de **Tomato Architecture**. El nombre no significa nada (igual que "hexagonal" tampoco); es un guiño para no tomarse la arquitectura tan en serio. Si quieres el detalle, está en [su blog](https://www.sivalabs.in/blog/tomato-architecture-pragmatic-approach-to-software-design/). Lo que nos importa acá: package-by-feature, servicios concretos, y abrazar el framework en vez de envolverlo en abstracciones.

## Parte 4 — Conectar Supabase

Abre tu proyecto de Supabase, ve a **Project Settings → Database** y copia los datos de conexión. Vamos a pasárselos a Spring por variables de entorno, no quemados en el código.

Crea o edita `src/main/resources/application.yaml`:

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
