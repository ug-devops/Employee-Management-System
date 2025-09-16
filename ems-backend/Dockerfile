# ==============================
# Build stage
# ==============================
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app

# Copy Maven wrapper and pom.xml first (dependency caching)
COPY mvnw* pom.xml ./
COPY .mvn .mvn

# Ensure mvnw is executable and download dependencies
RUN chmod +x mvnw && ./mvnw dependency:go-offline -B

# Copy source code
COPY src src

# Build application (skip tests for faster build)
RUN ./mvnw clean package -DskipTests

# ==============================
# Runtime stage
# ==============================
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Copy only the built jar from builder stage
COPY --from=builder /app/target/*.jar app.jar

# Expose backend port
EXPOSE 8080

# Run Spring Boot application
ENTRYPOINT ["java", "-jar", "app.jar"]

