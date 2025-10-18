FROM eclipse-temurin:8-jdk-jammy AS builder

WORKDIR /app

COPY .mvn/ .mvn

COPY mvnw pom.xml ./

RUN chmod +x mvnw

RUN ./mvnw -Dhttps.protocols=TLSv1.2 dependency:go-offline

COPY src ./src

RUN ./mvnw clean install -DskipTests

# Create a small image for running the app

FROM eclipse-temurin:8-jdk-jammy

WORKDIR /app

COPY --from=builder /app/target/*.jar target/users-api-0.0.1-SNAPSHOT.jar

EXPOSE 8083

ENTRYPOINT ["java", "-jar", "target/users-api-0.0.1-SNAPSHOT.jar"]
