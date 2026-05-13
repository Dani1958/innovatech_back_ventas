# ============================================
# Etapa 1: Build con Maven + JDK 17
# ============================================
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /build

# Cacheamos dependencias copiando primero el pom.xml
COPY pom.xml .
RUN mvn -B -q dependency:go-offline

# Copiamos el código fuente y empaquetamos el .jar (sin correr tests)
COPY src ./src
RUN mvn -B -q clean package -DskipTests \
 && mv target/*.jar target/app.jar

# ============================================
# Etapa 2: Runtime liviano con JRE 17
# ============================================
FROM eclipse-temurin:17-jre-alpine AS runtime

# Crear usuario no-root (mínimo privilegio)
RUN addgroup -S spring && adduser -S spring -G spring

WORKDIR /app

# Copiamos solo el .jar final desde la etapa anterior
COPY --from=builder /build/target/app.jar /app/app.jar

# Cambiamos la propiedad y bajamos privilegios
RUN chown -R spring:spring /app
USER spring

# Spring Boot por defecto escucha en 8080
EXPOSE 8080

# Permitir tuning de JVM por variable de entorno
ENV JAVA_OPTS=""

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
