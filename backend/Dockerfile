FROM maven:3.9.11-eclipse-temurin-21 AS maven

WORKDIR /app

COPY pom.xml .

COPY src ./src

RUN mvn clean package

# separate build and run to not have src code

FROM eclipse-temurin:25

WORKDIR /app

COPY --from=maven /app/target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
