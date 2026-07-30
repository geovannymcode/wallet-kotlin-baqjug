# Requisitos y setup

La idea de este taller es que nadie pierda 20 minutos peleando con infraestructura que no arranca. El grueso del stack corre en la nube y se configura desde el navegador: Supabase para Postgres y Redpanda para los eventos. Docker aparece solo en las fases finales —el correo de prueba con Mailpit (Fase 7) y la imagen de la app (Fase 10)—; lo demás es lo de siempre para programar en Kotlin.

## En tu máquina

| Herramienta | Versión | Para qué |
|-------------|---------|----------|
| **JDK** | 25 | Compilar y correr. Sirve Temurin, Zulu o el que uses |
| **IntelliJ IDEA** | Community o Ultimate | El IDE. Community alcanza de sobra |
| **Git** | cualquiera reciente | Clonar el repo y moverte entre fases |
| Un cliente REST | Bruno, Postman o `curl` | Probar los endpoints |
| **Docker** | reciente | Solo desde la Fase 7: el correo de prueba (Mailpit) y empaquetar la app |

La base y el broker viven en la nube, así que no montas nada local para eso. Docker no lo necesitas al arrancar: entra en juego recién en la Fase 7 (correo con Mailpit) y en la Fase 10 (despliegue). Si ya programas en Java o Kotlin, seguro tienes casi todo esto.

## En la nube (se configura en vivo)

### Supabase (la base de datos)

Supabase te da un Postgres gestionado detrás de un login con GitHub. Lo configuramos en la Fase 0.

1. Entra a [supabase.com](https://supabase.com) y crea un proyecto (el plan gratis alcanza).
2. Guarda la contraseña de la base que te pide al crear el proyecto.
3. En **Project Settings → Database** vas a encontrar la cadena de conexión. Esa es la que le pasamos a Spring.

### Redpanda (para la parte de eventos)

Redpanda es un broker compatible con el protocolo de Kafka, más liviano y sin Zookeeper. Para no depender de que cada quien tenga Docker corriendo, usamos **Redpanda Serverless**, que también vive en la nube y se configura desde el navegador. Lo hacemos en la Fase 6.

1. Entra a [redpanda.com](https://www.redpanda.com) y crea un clúster Serverless (tiene capa gratis).
2. Guarda el **bootstrap server** y las credenciales SASL (usuario y contraseña) que te genera.

!!! note "Alternativa local con Docker"
    Si tienes Docker a mano, puedes levantar Redpanda local con `rpk container start` en vez de la nube. La guía usa la versión Serverless por defecto para no depender de que cada quien tenga Docker, pero el código es el mismo; solo cambian las propiedades de conexión.

Con esto listo, arranca por la [Fase 0](fase-00-arranque.md).
