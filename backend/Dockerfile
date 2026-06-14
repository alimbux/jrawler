FROM maven:3.9.9-eclipse-temurin-21 AS build

WORKDIR /workspace

COPY pom.xml .
RUN mvn -B dependency:go-offline

COPY src ./src
RUN mvn -B package -DskipTests

FROM mcr.microsoft.com/playwright/java:v1.49.0-noble

WORKDIR /app

COPY --from=build /workspace/target/jrawler-backend-*.jar /app/app.jar

ENV SERVER_PORT=8080
ENV JAVA_OPTS=""

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
