FROM maven:3.9.9-eclipse-temurin-21 AS builder

WORKDIR /app

# Copy root pom and module poms first to leverage Docker layer caching
COPY pom.xml .
COPY patient-service/pom.xml patient-service/
COPY billing-service/pom.xml billing-service/
COPY analytics-service/pom.xml analytics-service/
COPY auth-service/pom.xml auth-service/
COPY api-gateway/pom.xml api-gateway/
COPY integration-tests/pom.xml integration-tests/

# Download all dependencies for all modules to cache them
RUN mvn dependency:go-offline -B

# Copy actual source code
COPY patient-service/src patient-service/src

# Build the specific module
RUN mvn clean package -pl patient-service -am -DskipTests

# Stage 2: Production Runner
FROM openjdk:21-jdk AS runner
WORKDIR /app

COPY --from=builder /app/patient-service/target/patient-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 4000

ENTRYPOINT ["java", "-jar", "app.jar"]
