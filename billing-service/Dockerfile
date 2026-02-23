FROM maven:3.9.9-eclipse-temurin-21 AS builder

WORKDIR /app

COPY pom.xml .
COPY patient-service/pom.xml patient-service/
COPY billing-service/pom.xml billing-service/
COPY analytics-service/pom.xml analytics-service/
COPY auth-service/pom.xml auth-service/
COPY api-gateway/pom.xml api-gateway/
COPY integration-tests/pom.xml integration-tests/

RUN mvn dependency:go-offline -B

# Copy Protobuf schemas and Java logic
COPY billing-service/src billing-service/src

# Maven will automatically trigger protoc generation during compilation
RUN mvn clean package -pl billing-service -am -DskipTests

FROM openjdk:21-jdk AS runner
WORKDIR /app

# Pull only the generated runner artifact
COPY --from=builder /app/billing-service/target/billing-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 4001 9001

ENTRYPOINT ["java", "-jar", "app.jar"]
