# Diagramas del taller (código Mermaid para Excalidraw)

Excalidraw importa Mermaid **nativo** solo para tres tipos: `flowchart`, `sequenceDiagram` y `classDiagram`. Cualquier otro tipo lo mete como imagen plana (no editable). Por eso este set usa solo esos tres, y a propósito **varía el tipo** para que los diagramas no se vean todos iguales.

**Cómo usarlos:** entra a [excalidraw.com](https://excalidraw.com/) → menú (☰) → **Mermaid to Excalidraw** → pega el bloque → *Insert* → estiliza y exporta como PNG.

!!! tip "Evita el glitch del `\n`"
    En el diagrama viejo, algunas cajas mostraban un `\n` literal. Eso pasa cuando el texto trae saltos de línea que Excalidraw no interpreta. Acá dejé todas las etiquetas en una sola línea a propósito. Si necesitas partir una etiqueta, hazlo en Excalidraw después de importar, no en el Mermaid.

---

## 1. Arquitectura general — *flowchart* (Fase 0 / inicio)

Reemplazo editable del diagrama "cliente → app → base".

```mermaid
flowchart LR
    C["Cliente / consumer REST"]
    subgraph App["Aplicación (wallet)"]
        API["REST · WalletBAQ"]
    end
    subgraph Datos["Base de datos"]
        DB[("Postgres")]
    end
    C -->|HTTP| API
    API -->|ORM / JPA| DB
```

---

## 2. Flujo de una transferencia — *sequence* (Fase 3 y 4)

Muestra el orden temporal y que todo pasa dentro de una transacción. (Tipo distinto: diagrama de secuencia.)

```mermaid
sequenceDiagram
    actor U as Cliente
    participant TC as TransferController (web)
    participant TS as TransferService (domain)
    participant AS as AccountService (domain)
    participant MS as MovementService (domain)
    U->>TC: POST /api/transfers
    TC->>TS: transfer(request)
    Note over TS,MS: todo dentro de @Transactional
    TS->>AS: moveMoney(from, to, amount)
    AS-->>TS: MoveResult.Success
    TS->>MS: record(from, to, amount)
    MS-->>TS: ok
    TC-->>U: 200 COMPLETED
```

---

## 3. Modelo por feature (account) — *class* (Fase 2 y 3)

Las clases clave y cómo se relacionan: entidad vs DTO, el mapper, el servicio concreto, el repositorio como interfaz. (Tipo distinto: diagrama de clases.)

```mermaid
classDiagram
    class AccountEntity {
        +UUID id
        +String owner
        +String email
        +BigDecimal balance
        +Instant createdAt
        +Instant updatedAt
    }
    class AccountResponse {
        +UUID id
        +String owner
        +String email
        +BigDecimal balance
    }
    class AccountService {
        +getById(id) AccountResponse
        +moveMoney(from, to, amount) MoveResult
    }
    class AccountMapper {
        +toResponse(entity) AccountResponse
    }
    class AccountRepository {
        <<interface>>
    }
    class MoveResult {
        <<sealed>>
    }
    AccountService --> AccountRepository : usa
    AccountService --> AccountMapper : usa
    AccountService ..> MoveResult : devuelve
    AccountMapper --> AccountEntity : lee
    AccountMapper --> AccountResponse : produce
    AccountRepository --> AccountEntity : gestiona
```

---

## 4. Pub/Sub de eventos — *flowchart* (Fase 5 y 7)

Un evento, dos grupos de consumidores. Cada grupo recibe su copia.

```mermaid
flowchart LR
    T["transfer (producer)"] -->|"publica MovimientoRegistrado"| K{{"Redpanda — topic wallet.movements"}}
    K -->|"grupo notification"| N["notification (envía correo)"]
    K -->|"grupo movement"| M["movement (guarda el registro)"]
```

---

## 5. Patrón outbox — *sequence* (Fase 8)

El evento y el dato se guardan juntos en una transacción; un relay aparte publica. (Secuencia con `loop`.)

```mermaid
sequenceDiagram
    participant TS as TransferService
    participant DB as Postgres
    participant R as OutboxRelay
    participant K as Redpanda
    Note over TS,DB: una sola transacción
    TS->>DB: mover saldo + guardar fila en outbox
    loop cada 2s
        R->>DB: lee filas no enviadas (FOR UPDATE SKIP LOCKED)
        R->>K: publica el evento
        R->>DB: marca sent_at
    end
```

---

## 6. Coroutines: llamadas en paralelo — *sequence* (Fase 9)

El consumidor llama a tres servicios externos a la vez con `async`/`awaitAll`. (Secuencia con bloque `par`.)

```mermaid
sequenceDiagram
    participant L as Listener (suspend)
    participant NS as NotificationService
    participant E as EmailClient
    participant P as PushClient
    participant A as AntifraudClient
    L->>NS: notificar(evento)
    Note over NS,A: coroutineScope { async ... }
    par en paralelo
        NS->>E: enviar()
    and
        NS->>P: enviar()
    and
        NS->>A: revisar()
    end
    E-->>NS: ok
    P-->>NS: ok
    A-->>NS: ok
    Note over NS: awaitAll — ~200 ms, no 600 ms
```
