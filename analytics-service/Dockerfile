FROM maven:3.9.9-eclipse-temurin-21 AS builder

WORKDIR /app

COPY pom.xml .
COPY patient-service/pom.xml patient-service/
COPY billing-service/pom.xml billing-service/
COPY analytics-service/pom.xml analytics-service/
COPY auth-service/pom.xml auth-service/
COPY api-gateway/pom.xml api-gateway/
COPY integration-tests/pom.xml integration-tests/

RUN mvn dependency:go-offline -B -pl analytics-service -am

# Copy Protobuf Event schemas and Java logic
COPY analytics-service/src analytics-service/src

# Maven will generate Kafka Protobuf stubs here
RUN mvn clean package -pl analytics-service -am -DskipTests

FROM openjdk:21-jdk AS runner
WORKDIR /app

COPY --from=builder /app/analytics-service/target/analytics-service-0.0.1-SNAPSHOT.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
