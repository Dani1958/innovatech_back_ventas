# Innovatech — Backend Ventas

API REST del microservicio de **Ventas** de Innovatech Chile, construida con Spring Boot 3 y MySQL. Contenedorizada con Docker y desplegada automáticamente en AWS EC2 mediante GitHub Actions.

Forma parte del proyecto de **Evaluación Parcial N°2 — Introducción a Herramientas DevOps (ISY1101)**.

---

## Tabla de contenidos

1. [Arquitectura general](#arquitectura-general)
2. [Stack técnico](#stack-técnico)
3. [Estructura del repositorio](#estructura-del-repositorio)
4. [Ejecución local sin Docker](#ejecución-local-sin-docker)
5. [Ejecución local con Docker](#ejecución-local-con-docker)
6. [Variables de entorno](#variables-de-entorno)
7. [Endpoints principales](#endpoints-principales)
8. [Persistencia de datos](#persistencia-de-datos)
9. [Pipeline CI/CD (GitHub Actions)](#pipeline-cicd-github-actions)
10. [Despliegue en AWS EC2](#despliegue-en-aws-ec2)
11. [Decisiones técnicas](#decisiones-técnicas)

---

## Arquitectura general

```
┌────────────────────┐         ┌────────────────────┐         ┌────────────────────┐
│  EC2 Frontend       │  HTTPS │  EC2 Back-Ventas    │  JDBC  │     AWS RDS         │
│  (pública, :80)     │ ─────► │  (privada, :8080)   │ ─────► │     MySQL           │
│  nginx + SPA        │        │  Spring Boot        │        │     ventas_db       │
└────────────────────┘        └────────────────────┘        └────────────────────┘
```

Este servicio vive en una EC2 **privada** y solo es alcanzable desde el Security Group del frontend. La persistencia se delega a un MySQL en **AWS RDS**.

---

## Stack técnico

| Capa | Tecnología |
|---|---|
| Lenguaje | Java 17 |
| Framework | Spring Boot 3.4.4 |
| Persistencia | Spring Data JPA + Hibernate |
| Base de datos | MySQL 8 (AWS RDS); H2 en tests |
| Documentación API | SpringDoc OpenAPI 2.7 (Swagger UI) |
| Build | Maven 3.9 |
| Contenedorización | Docker (multi-stage build) |
| Imagen runtime | `eclipse-temurin:17-jre-alpine` |
| CI/CD | GitHub Actions |
| Registry | Docker Hub |
| Cloud | AWS EC2 + RDS |

---

## Estructura del repositorio

```
back-Ventas_SpringBoot/
├── Springboot-API-REST/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/citt/
│   │   │   │   ├── config/                  # OpenApiConfig
│   │   │   │   ├── controller/              # VentaController
│   │   │   │   ├── persistence/
│   │   │   │   │   ├── entity/              # Venta.java
│   │   │   │   │   ├── repository/          # VentaRepository
│   │   │   │   │   └── services/            # VentaService + Impl
│   │   │   │   └── exceptions/              # Manejo global de errores
│   │   │   └── resources/
│   │   │       ├── application.properties
│   │   │       └── application-test.properties
│   │   └── test/
│   │       └── java/persistence/service/
│   │           └── VentaServiceTest.java
│   ├── pom.xml
│   ├── Dockerfile                           # Multi-stage maven → JRE
│   ├── .dockerignore
│   └── mvnw / mvnw.cmd
├── docker-compose.deploy.yml                # Compose de PRODUCCIÓN (EC2)
├── .github/
│   └── workflows/
│       └── deploy.yml                       # Pipeline CI/CD
└── README.md
```

---

## Ejecución local sin Docker

Requiere Java 17 y un MySQL local en `localhost:3306`.

```bash
cd Springboot-API-REST

export DB_ENDPOINT=localhost
export DB_PORT=3306
export DB_NAME=ventas_db
export DB_USERNAME=root
export DB_PASSWORD=tu_password

./mvnw spring-boot:run
```

Swagger UI: http://localhost:8080/swagger-ui.html

### Correr tests

```bash
./mvnw test
```

Los tests usan H2 en memoria (`application-test.properties`), no requieren MySQL.

---

## Ejecución local con Docker

```bash
cd Springboot-API-REST

docker build -t innovatech-back-ventas:dev .

docker run --rm -p 8080:8080 \
  -e DB_ENDPOINT=host.docker.internal \
  -e DB_PORT=3306 \
  -e DB_NAME=ventas_db \
  -e DB_USERNAME=root \
  -e DB_PASSWORD=tu_password \
  innovatech-back-ventas:dev
```

Para el **stack completo** (frontend + ambos backends + MySQL en contenedor), usar el `docker-compose.yml` de la raíz `proyecto semestral/`.

---

## Variables de entorno

| Variable | Descripción | Ejemplo |
|---|---|---|
| `DB_ENDPOINT` | Host del MySQL (RDS endpoint o nombre de servicio en compose) | `xxx.rds.amazonaws.com` |
| `DB_PORT` | Puerto MySQL | `3306` |
| `DB_NAME` | Nombre de la base de datos | `ventas_db` |
| `DB_USERNAME` | Usuario MySQL | `innovatech` |
| `DB_PASSWORD` | Contraseña MySQL | `xxxxxx` |
| `JAVA_OPTS` | Tuning de la JVM (opcional) | `-Xms256m -Xmx512m` |

La JDBC URL del [`application.properties`](./Springboot-API-REST/src/main/resources/application.properties) incluye `createDatabaseIfNotExist=true`, por lo que Hibernate crea la base si no existe.

---

## Endpoints principales

Una vez levantado, la documentación interactiva está disponible en:

- **Swagger UI:** `http://<host>:8080/swagger-ui.html`
- **OpenAPI JSON:** `http://<host>:8080/v3/api-docs`

El controlador principal vive en [`VentaController.java`](./Springboot-API-REST/src/main/java/com/citt/controller/VentaController.java).

---

## Persistencia de datos

### En producción (EC2 + RDS)

La base de datos vive en **AWS RDS** (MySQL 8). El contenedor del backend solo lee/escribe vía JDBC. RDS gestiona backups, snapshots y alta disponibilidad.

### En desarrollo local (docker-compose)

Cuando se levanta el stack completo, MySQL corre como contenedor con un **volumen Docker named** (`mysql_data`) montado en `/var/lib/mysql`:

- Los datos sobreviven a `docker compose down` y reinicios del host.
- El volumen es portable entre máquinas (no depende de paths absolutos del host).

**Por qué named volume y no bind mount:** un *bind mount* obligaría a depender de una ruta específica del sistema operativo (distinta en Windows, Mac y Linux), mientras que un *named volume* lo gestiona Docker de manera uniforme. Para datos de base de datos es la opción recomendada.

---

## Pipeline CI/CD (GitHub Actions)

Archivo: [.github/workflows/deploy.yml](./.github/workflows/deploy.yml)

**Trigger:** `push` a la rama `deploy`.

**Flujo:**

```
push → build (multi-stage) → push a Docker Hub → SSH a EC2 → pull + restart
```

**Pasos:**

1. **Checkout** del código.
2. **Login a Docker Hub** con `DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN`.
3. **Setup Buildx** para caché remota de capas.
4. **Build & Push** con dos tags:
   - `latest` (móvil)
   - `${{ github.sha }}` (inmutable, para trazabilidad y rollback)
5. **SSH a la EC2 Back-Ventas** y ejecuta:
   ```bash
   docker compose -f docker-compose.deploy.yml pull
   docker compose -f docker-compose.deploy.yml up -d --remove-orphans
   docker image prune -f
   ```

### Secrets necesarios

| Secret | Descripción |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub |
| `DOCKERHUB_TOKEN` | Access Token (no la contraseña) |
| `EC2_VENTAS_HOST` | IP/DNS de la EC2 de Ventas |
| `EC2_USER` | `ubuntu` o `ec2-user` |
| `EC2_SSH_KEY` | Contenido del `.pem` |
| `DB_ENDPOINT` | Endpoint del RDS |
| `DB_PORT` | `3306` |
| `DB_NAME_VENTAS` | `ventas_db` |
| `DB_USERNAME` | Usuario RDS |
| `DB_PASSWORD` | Password RDS |

---

## Despliegue en AWS EC2

### 1. Preparar la EC2 (una sola vez)

```bash
sudo apt-get update -y
sudo apt-get install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker

mkdir -p ~/innovatech-back-ventas
cd ~/innovatech-back-ventas
# Subir docker-compose.deploy.yml o clonar este repo
```

### 2. Security Group recomendado

| Puerto | Origen | Justificación |
|---|---|---|
| 8080 | Security Group del frontend | Solo el front puede consumir la API |
| 22 | Tu IP fija | SSH administrativo |

### 3. RDS

| Configuración | Valor |
|---|---|
| Motor | MySQL 8 |
| Security Group | Permitir 3306 desde SG de los backends |
| Acceso público | **No** |

### 4. Disparar el despliegue

```bash
git checkout -b deploy
git push origin deploy
```

---

## Decisiones técnicas

### Multi-stage build (Maven → JRE)
**Por qué:** la etapa de build necesita ~600 MB (Maven + JDK + dependencias). El runtime solo necesita la JRE (~180 MB) y el `.jar` final. Resultado: imagen final ~230 MB en vez de ~700 MB.

### Usuario no-root (`USER spring`)
**Por qué:** principio de mínimo privilegio. Si la app es comprometida, el atacante no tiene root dentro del contenedor.

### `eclipse-temurin:17-jre-alpine` como runtime
**Por qué:** alpine reduce el tamaño base ~100 MB; `jre` (no `jdk`) porque la app ya está compilada.

### Variables de entorno para credenciales
**Por qué:** las credenciales no deben estar en el repo. `application.properties` usa `${DB_ENDPOINT}`, `${DB_PASSWORD}`, etc., y Spring las resuelve en runtime desde el entorno del contenedor. En CI/CD las inyecta GitHub Actions desde sus Secrets.

### MySQL en RDS (no en contenedor) en producción
**Por qué:** RDS provee backups automáticos, snapshots, alta disponibilidad y parchado del motor. Un MySQL en contenedor sirve para desarrollo, no para producción.

### H2 en tests
**Por qué:** los tests de integración corren en GitHub Actions sin red, sin Docker. H2 en memoria es rápido, portable y suficiente para verificar mapeos JPA y lógica de servicio. Configurado en `application-test.properties`.

### Tag `${{ github.sha }}` en cada imagen
**Por qué:** cada commit produce una imagen inmutable. Si un deploy rompe algo, podemos `docker pull <user>/back-ventas:<SHA-anterior>` y restaurar en segundos.

### Rama `deploy` como gate
**Por qué:** mantener `main` como rama de integración y `deploy` como promoción a producción. Solo merges explícitos a `deploy` desencadenan despliegues.

---

## Información del curso

| | |
|---|---|
| Asignatura | ISY1101 — Introducción a Herramientas DevOps |
| Evaluación | EP2 (30%) |
| Institución | Duoc UC |
| Año | 2025 |
