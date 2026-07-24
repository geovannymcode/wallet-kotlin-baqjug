# Fase 10 · Despliegue con GitHub Actions

**Rama**: opcional, es el paso de "shipping". Hasta acá corriste todo en tu máquina; esta fase lleva la wallet a la nube y hace que cada `push` a `main` la construya, la teste y la despliegue sola.
**Lo que vas a lograr**: empaquetar la app en una imagen Docker, montar un pipeline de GitHub Actions que compila y testea, y desplegar a Render (o Railway) con `push` a `main`.

!!! warning "Esta fase es de operaciones (ops), no de código de negocio"
    Nada del código de negocio cambia. Acá montamos lo que falta para que la wallet viva fuera de tu laptop: imagen Docker, integración continua y despliegue. Es opcional, pero es lo que separa "me corre en local" de "está en internet".

!!! note "El correo ya lo montamos en la Fase 7"
    En la Fase 7, `notification` manda el correo con Mailpit en local. Al desplegar, las variables `SMTP_*` dejan de apuntar a Mailpit y apuntan a un proveedor real (SendGrid, Amazon SES, Mailgun). El código no cambia; solo las variables.

---

## Parte 1 — Empaquetar la app en una imagen Docker

Para que la wallet corra en cualquier lado igual que en tu máquina, la metemos en una **imagen Docker**: un paquete con la app y todo lo que necesita para arrancar.

!!! abstract "Concepto al paso: imagen Docker y build multi-etapa"
    Una imagen Docker es una plantilla con tu app lista para correr. La construimos en **dos etapas**: la primera tiene el JDK completo y compila el `jar`; la segunda, mucho más liviana, solo tiene el JRE y el `jar` ya compilado. Así la imagen final no carga con el compilador ni el código fuente: pesa menos y arranca más rápido.

```dockerfile title="Dockerfile"
# Etapa 1: compilar
FROM eclipse-temurin:25-jdk AS build
WORKDIR /app
COPY . .
RUN ./gradlew bootJar --no-daemon

# Etapa 2: correr (imagen liviana, solo JRE)
FROM eclipse-temurin:25-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

!!! note "Alternativa sin Dockerfile"
    Spring Boot puede armar la imagen solo, con `./gradlew bootBuildImage` (usa buildpacks, sin escribir un Dockerfile). Cualquiera de las dos sirve; el Dockerfile es más explícito y portable entre hosts, por eso lo mostramos.

## Parte 2 — El pipeline de integración continua (GitHub Actions)

Queremos que cada cambio se compile y se testee solo, antes de desplegar nada.

!!! abstract "Concepto al paso: CI/CD y GitHub Actions"
    **CI** (integración continua) es automatizar que, con cada `push`, se compile y se corran los tests. **CD** (despliegue continuo) es que, si eso pasa, se despliegue solo. **GitHub Actions** ejecuta esos pasos en un *runner* (una máquina que GitHub te presta) según recetas en `.github/workflows/`. Cada receta es un *workflow*.

```yaml title=".github/workflows/ci.yml"
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configurar JDK 25
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '25'

      - name: Compilar y testear
        run: ./gradlew test
```

!!! abstract "Spring al paso: por qué los tests corren sin infra"
    ¿Te acuerdas del test de la Fase 4 con MockK? Corre sin base ni broker, así que el runner de GitHub lo ejecuta sin levantar Supabase ni Redpanda. Los tests que sí necesitan infra real (con Testcontainers) también corren en Actions, porque el runner tiene Docker; eso ya es un paso más avanzado.

## Parte 3 — Desplegar a Render (o Railway)

Render y Railway despliegan tu imagen Docker directo desde el repo. La forma más simple para un taller es dejar que **el propio host construya y despliegue** con cada `push` a `main`:

1. En Render, creas un **Web Service** y conectas tu repo de GitHub.
2. Eliges **Docker** como entorno: Render detecta tu `Dockerfile` y construye la imagen.
3. En **Environment**, pones las variables (las mismas que usas en local, con los valores de la nube).
4. Cada `push` a `main` dispara un build y un deploy automáticos.

Así queda un reparto limpio: **GitHub Actions hace el CI** (compila y testea) y **Render hace el CD** (construye la imagen y la despliega). No necesitas meter secretos de despliegue en Actions.

!!! note "Si quieres disparar el deploy desde Actions"
    Render te da un *Deploy Hook* (una URL secreta). Si prefieres que el deploy ocurra solo después de que pasen los tests, guardas esa URL como secreto en GitHub y agregas un paso al final del workflow:

    ```yaml
    - name: Desplegar en Render
      if: github.ref == 'refs/heads/main'
      run: curl -fsSL "$RENDER_DEPLOY_HOOK"
      env:
        RENDER_DEPLOY_HOOK: ${{ secrets.RENDER_DEPLOY_HOOK }}
    ```

!!! warning "La UI de estos hosts cambia seguido"
    Los pasos exactos de Render/Railway (nombres de botones, capa gratis) cambian con el tiempo. Toma esto como el mapa —conectar repo, elegir Docker, poner variables, deploy con push— y confirma el detalle en su documentación al momento de hacerlo.

## Parte 4 — Variables de entorno y secretos, en un solo lugar mental

El código nunca sabe un secreto; lo lee del entorno. Lo que cambia es **quién** pone esas variables en cada lado:

- **En tu máquina**: en IntelliJ (**Run → Edit Configurations → Environment variables**).
- **En GitHub Actions**: en **Settings → Secrets and variables → Actions** del repo.
- **En Render/Railway**: en la sección **Environment** del servicio.

Las que necesita la wallet:

| Variable | Para qué |
|----------|----------|
| `SUPABASE_DB_URL` / `SUPABASE_DB_USER` / `SUPABASE_DB_PASSWORD` | La base Postgres |
| `REDPANDA_BOOTSTRAP` / `REDPANDA_USER` / `REDPANDA_PASSWORD` | El broker de eventos |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASSWORD` | El correo (Mailpit en local, un proveedor real en la nube) |

## Cierre de la fase

Con esto, la wallet dejó de vivir solo en tu laptop: cada `push` a `main` pasa por CI (compila y testea), se construye la imagen Docker, y Render la despliega. Corre en la nube, con su base en Supabase, sus eventos en Redpanda, y correos que en local atrapa Mailpit y en producción salen por un SMTP real.

Y con eso cerramos el taller: arrancaste con un endpoint REST contra una base, y terminaste con una wallet transaccional, desacoplada por eventos, testeada, empaquetada y desplegada. Nada de humo: código y tuberías de verdad.

Revisa las [Referencias](referencias.md) para seguir por tu cuenta.
