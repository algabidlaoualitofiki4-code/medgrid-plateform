FROM maven:3.9.9-eclipse-temurin-17-alpine AS build
WORKDIR /workspace
COPY pom.xml .
COPY contracts ./contracts
COPY services/audit-service ./services/audit-service
RUN mvn -pl services/audit-service -am clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S medgrid && adduser -S medgrid -G medgrid
COPY --from=build /workspace/services/audit-service/target/*.jar app.jar
USER medgrid
EXPOSE 8088
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
