FROM maven:3.9.9-eclipse-temurin-21 AS builder

WORKDIR /app

COPY pom.xml .
COPY patient-service/pom.xml patient-service/
COPY billing-service/pom.xml billing-service/
COPY analytics-service/pom.xml analytics-service/
COPY auth-service/pom.xml auth-service/
COPY api-gateway/pom.xml api-gateway/
COPY integration-tests/pom.xml integration-tests/

RUN mvn dependency:go-offline -B -pl auth-service -am

COPY auth-service/src auth-service/src

RUN mvn clean package -pl auth-service -am -DskipTests

FROM openjdk:21-jdk AS runner
WORKDIR /app

COPY --from=builder /app/auth-service/target/auth-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 4005

ENTRYPOINT ["java", "-jar", "app.jar"]
