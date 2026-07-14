# Fase 1 · El dominio de la wallet

**Rama**: `fase-1`
**Lo que vas a lograr**: modelar la cuenta con su saldo como entidad JPA, crear la primera migración de Flyway, y ver la app arrancar contra Supabase con datos de prueba.

---

## Parte 1 — La cuenta como entidad (15 min)

Arrancamos por la feature `account`, en su `internal`. Una cuenta tiene un id, un dueño y un saldo. El saldo es lo delicado: es plata, y la plata no se representa con `Double`.

```kotlin title="account/internal/Account.kt"
package com.baqjug.wallet.account.internal

import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.Id
import jakarta.persistence.Table
import java.math.BigDecimal
import java.util.UUID

@Entity
@Table(name = "accounts")
class Account(
    @Column(nullable = false)
    val owner: String,

    @Column(nullable = false, precision = 19, scale = 2)
    var balance: BigDecimal = BigDecimal.ZERO,

    @Id
    val id: UUID = UUID.randomUUID()
)
```

!!! abstract "Kotlin al paso: `class`, `val`, `var` y el constructor"
    En Kotlin declaras la clase y su constructor en una sola línea. Cada parámetro con `val` o `var` es además una propiedad de la clase. `val` es inmutable (como `final` en Java): `owner` e `id` no cambian nunca. `var` es mutable: `balance` sí cambia, porque el saldo sube y baja. El `= BigDecimal.ZERO` y el `= UUID.randomUUID()` son **valores por defecto**: si no los pasas, se usan esos.

!!! abstract "Spring al paso: `@Entity`, `@Table`, `@Id`, `@Column`"
    `@Entity` le dice a JPA que esta clase se guarda en una tabla. `@Table(name = "accounts")` fija el nombre de la tabla. `@Id` marca la llave primaria. `@Column` describe la columna: `nullable = false` la hace obligatoria, y `precision = 19, scale = 2` define un número con 19 dígitos y 2 decimales, exacto, para la plata.

!!! danger "Nunca uses `Double` para dinero"
    `Double` es de punto flotante: `0.1 + 0.2` no da `0.3`, da `0.30000000000000004`. En un saldo eso es un desastre. `BigDecimal` guarda el valor exacto. Para dinero, siempre `BigDecimal` con una escala fija.

## Parte 2 — El repositorio (5 min)

El repositorio es cómo leemos y guardamos cuentas. Con Spring Data JPA no lo implementas: declaras una interfaz y Spring te da la implementación.

```kotlin title="account/internal/AccountRepository.kt"
package com.baqjug.wallet.account.internal

import org.springframework.data.jpa.repository.JpaRepository
import java.util.UUID

interface AccountRepository : JpaRepository<Account, UUID>
```

!!! abstract "Kotlin al paso: `interface`"
    Una `interface` es un contrato: dice qué operaciones existen, sin decir cómo se hacen. Acá ni siquiera escribimos métodos. Al extender `JpaRepository<Account, UUID>` heredas gratis `save`, `findById`, `findAll`, `deleteById` y varios más. `Account` es el tipo que maneja y `UUID` el tipo de su id.

!!! abstract "Spring al paso: cómo aparece la implementación"
    Tú declaras la interfaz y Spring Data, en tiempo de arranque, genera la clase que la implementa y la registra como un bean listo para inyectar. Es una de las cosas que más tiempo te ahorra en Spring: los repositorios básicos no se escriben.

## Parte 3 — La primera migración de Flyway (10 min)

Flyway aplica migraciones SQL versionadas en orden, y lleva registro de cuáles ya corrió. Así el esquema de la base es reproducible y viaja en el repo.

Crea la carpeta `src/main/resources/db/migration` y adentro:

```sql title="db/migration/V1__create_accounts.sql"
CREATE TABLE accounts (
    id      UUID           NOT NULL,
    owner   VARCHAR(255)   NOT NULL,
    balance NUMERIC(19, 2) NOT NULL DEFAULT 0,
    CONSTRAINT pk_accounts PRIMARY KEY (id)
);

-- Dos cuentas de prueba para poder transferir en las siguientes fases.
INSERT INTO accounts (id, owner, balance) VALUES
    ('11111111-1111-1111-1111-111111111111', 'Ana',  100000.00),
    ('22222222-2222-2222-2222-222222222222', 'Beto',      0.00);
```

!!! abstract "Concepto al paso: qué es una migración"
    Una migración es un script SQL con un cambio al esquema, numerado. El nombre `V1__create_accounts.sql` sigue la convención de Flyway: `V` de versión, el número, dos guiones bajos, y una descripción. Flyway las corre en orden la primera vez y anota en una tabla suya cuáles ya aplicó. La próxima vez no las repite. Nunca edites una migración ya aplicada: creas una nueva.

!!! note "El `NUMERIC(19, 2)` de la tabla concuerda con el `precision/scale` de la entidad"
    Como pusimos `ddl-auto=validate`, si la columna y la entidad no concuerdan, la app no arranca y te avisa. Es una red de seguridad: te obliga a mantener migración y entidad sincronizadas.

## Parte 4 — Arrancar la app (5 min)

Con las variables de entorno de Supabase configuradas, corre la aplicación desde IntelliJ (el botón de play sobre `WalletApplication`). Deberías ver en el log que Flyway aplica `V1`, y luego que la app queda escuchando.

Si abres Supabase, en el **Table Editor** ya aparecen la tabla `accounts` con Ana y Beto, y la tabla de control de Flyway.

## Cierre de la fase

```bash
./gradlew compileKotlin
git add .
git commit -m "fase-1: entidad Account, repositorio y migración inicial con Flyway"
git branch fase-1
```

Ya tienes cuentas con saldo en la base. En la [Fase 2](fase-02-rest.md) las exponemos por REST para consultar el saldo desde afuera.
