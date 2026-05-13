# Innovatech — Backend Ventas

API REST del microservicio de **Ventas** de Innovatech Chile, construida con Spring Boot 3 y MySQL. Contenedorizada con Docker y desplegada automáticamente en AWS EC2 mediante GitHub Actions.

Forma parte del proyecto de **Evaluación Parcial N°2 — Introducción a Herramientas DevOps (ISY1101)**.

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

## Persistencia de datos

### En producción (EC2 + RDS)

La base de datos vive en **AWS RDS** (MySQL 8). El contenedor del backend solo lee/escribe vía JDBC. RDS gestiona backups, snapshots y alta disponibilidad.

### En desarrollo local (docker-compose)

Cuando se levanta el stack completo, MySQL corre como contenedor con un **volumen Docker named** (`mysql_data`) montado en `/var/lib/mysql`:

- Los datos sobreviven a `docker compose down` y reinicios del host.
- El volumen es portable entre máquinas (no depende de paths absolutos del host).

**Por qué named volume y no bind mount:** un *bind mount* obligaría a depender de una ruta específica del sistema operativo (distinta en Windows, Mac y Linux), mientras que un *named volume* lo gestiona Docker de manera uniforme. Para datos de base de datos es la opción recomendada.

---

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

## Información del curso

| | |
|---|---|
| Asignatura | ISY1101 — Introducción a Herramientas DevOps |
| Evaluación | EP2 (30%) |
| Institución | Duoc UC |
| Año | 2025 |
