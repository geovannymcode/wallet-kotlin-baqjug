<div align="center">

# 💸 Una wallet en Kotlin: de un endpoint REST a eventos

**Un taller práctico donde construyes una billetera digital con Kotlin y Spring Boot — desde un `GET` de saldo hasta eventos que otros servicios consumen. Explicado desde cero, sin humo.**

[![Kotlin](https://img.shields.io/badge/Kotlin-2.3-7F52FF?logo=kotlin&logoColor=white)](https://kotlinlang.org)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-4.1-6DB33F?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![JDK](https://img.shields.io/badge/JDK-25-orange?logo=openjdk&logoColor=white)](https://adoptium.net)
[![Redpanda](https://img.shields.io/badge/Redpanda-Kafka_API-E24E42?logo=apachekafka&logoColor=white)](https://redpanda.com)
[![Docs](https://img.shields.io/badge/docs-MkDocs_Material-526CFE?logo=materialformkdocs&logoColor=white)](https://geovannymcode.github.io/wallet-kotlin-baqjug/)

### 📖 [Leer la guía online →](https://geovannymcode.github.io/wallet-kotlin-baqjug/)

</div>

---

## ¿De qué va esto?

Esta guía te lleva de la mano a construir una **wallet** de verdad: consultar saldo, transferir plata entre cuentas validando fondos, y registrar cada movimiento. Arranca como un endpoint REST directo contra Postgres y termina con ese mismo movimiento viajando como un **evento** que otro servicio consume por un broker.

No es un "hola mundo". Manejamos dinero con `BigDecimal`, transferencias transaccionales que **no dejan saldos rotos**, y desacople real por mensajería. Y lo explicamos **todo desde la base**: cada cosa de Kotlin y cada anotación de Spring se explica la primera vez que aparece. No asumimos que ya sabes.

> 💡 Se puede seguir sola, de principio a fin. Si nunca tocaste Kotlin o nunca hiciste Spring, tranquilo: para eso están las cajas **"Concepto al paso"**.

## 🧭 El recorrido (11 fases)

| Fase | Qué construyes |
|------|----------------|
| **0 · Arranque** | El problema, la historia de usuario, la arquitectura y el andamiaje |
| **1 · Dominio** | La cuenta con su saldo (JPA + Flyway), email y campos de auditoría |
| **2 · REST** | Consultar el saldo por HTTP (DTO + mapper + servicio concreto) |
| **3 · Transferencia** | Mover plata validando saldo, dentro de una transacción |
| **4 · Movimientos** | Registrar cada movimiento + test con MockK |
| **5 · Por qué eventos** | Broker, topic, producer, consumer, grupos y offsets |
| **6 · Publicar** | `transfer` publica el evento en Redpanda |
| **7 · Consumir** | `notification` (correo con Mailpit) y `movement` consumen el evento |
| **8 · De demo a producción** | Outbox, idempotencia y DLQ — con cajas *nivel senior* |
| **9 · Coroutines** | Un consumidor no bloqueante *(avanzada y opcional)* |
| **10 · Despliegue** | Docker + GitHub Actions + deploy a la nube |

Las Fases **0–7** son el hilo principal (de REST a eventos). La **8** endurece para producción, la **9** es un extra avanzado, y la **10** lo lleva todo a la nube. Puedes parar en la 7 con una wallet completa, o seguir hasta donde quieras.

## 🍅 Arquitectura: Tomato, sin sobre-ingeniería

Nada de hexagonal ni abstracciones "por si algún día". Seguimos **[Tomato Architecture](https://www.sivalabs.in/blog/tomato-architecture-pragmatic-approach-to-software-design/)** de Siva Prasad Reddy: código simple, aburrido y que sobrevive años.

- **Package by feature**: `account`, `transfer`, `movement`, `notification` — todo lo de una cosa, junto.
- Dentro de cada feature, **`domain`** (la lógica) y **`web`** / **`messaging`** (las puertas de entrada).
- **Servicios como clases concretas**, sin `interface` + `Impl` "por si acaso".
- Un **mapper** entre la entidad JPA y el DTO que sale por la API.

```
com.baqjug.wallet
├── account
│   ├── domain      # entidad, repositorio, servicio, mapper, DTOs, excepciones
│   └── web         # el controlador REST
├── transfer
│   ├── domain      # orquesta el caso de uso
│   ├── web
│   └── messaging   # publica el evento
├── movement
│   ├── domain
│   └── messaging   # consume el evento y guarda
├── notification
│   └── messaging   # consume el evento y notifica (correo)
└── web/exception   # manejo de errores compartido
```

## 🧰 Stack

**Kotlin 2.3** · **Spring Boot 4.1** · **JDK 25** · Spring Data JPA · Flyway · Spring for Apache Kafka · **Supabase** (PostgreSQL 17) · **Redpanda** · Docker (Mailpit + imagen) · GitHub Actions

La infraestructura pesada corre en la nube y se configura desde el navegador (Supabase y Redpanda Serverless), así que **no montas nada local** salvo lo de siempre para programar en Kotlin. Docker entra solo en las fases finales.

## 📖 Cómo leer la guía

**Online (lo más fácil):** 👉 **https://geovannymcode.github.io/wallet-kotlin-baqjug/**

**En local**, si quieres servir el sitio en tu máquina:

```bash
git clone https://github.com/geovannymcode/wallet-kotlin-baqjug.git
cd wallet-kotlin-baqjug

python3 -m venv venv && source venv/bin/activate
pip install mkdocs-material

mkdocs serve   # abre http://127.0.0.1:8000
```

## ✅ Requisitos para hacer el taller

**En tu máquina:** JDK 25 · IntelliJ IDEA (Community alcanza) · Git · un cliente REST (Bruno, Postman o `curl`) · Docker *(solo desde la Fase 7: Mailpit y la imagen)*.

**En la nube (gratis, se configura en vivo):** Supabase (Postgres) y Redpanda Serverless (broker de eventos).

## 🗂️ Estructura del repo

```
.
├── docs/            # la guía: un Markdown por fase + imágenes
│   └── img/         # los diagramas (Excalidraw)
├── mkdocs.yml       # configuración del sitio
└── README.md
```

> Este repo es la **guía**. El código de la wallet lo vas construyendo tú, fase por fase, siguiendo cada paso.

## 👨‍💻 Autores

**Geovanny Mendoza** — Senior Backend Engineer · Vaadin Champion · líder de **BAQJUG** (Barranquilla Java User Group).

[🌐 geovannycode.com](https://www.geovannycode.com) · [💼 LinkedIn](https://www.linkedin.com/in/geovannycode) · [🐦 @geovannycode](https://twitter.com/geovannycode) · [🐙 GitHub](https://github.com/geovannymcode)

**Maicol Ruidiaz** — Arquitecto de Software · Co-líder de **BAQJUG** (Barranquilla Java User Group).

[💼 LinkedIn](https://www.linkedin.com/in/maicol-r-8365a4b1/) · [🐦 @macavick](https://x.com/macavick) · [🐙 GitHub](https://github.com/mruidiazb)

## 📄 Licencia y uso

Material educativo de Geovanny Mendoza. Úsalo, apréndelo y compártelo; si lo reusas en un taller o una charla, dale crédito. 🙌

---

<div align="center">

Hecho con ☕ y Kotlin en Barranquilla. Si te sirve, dale una ⭐.

</div>
