FROM maven:3.9.9-eclipse-temurin-17-alpine AS build
WORKDIR /workspace
COPY pom.xml .
COPY contracts ./contracts
COPY services/admission-service ./services/admission-service
RUN mvn -pl services/admission-service -am clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S medgrid && adduser -S medgrid -G medgrid
COPY --from=build /workspace/services/admission-service/target/*.jar app.jar
USER medgrid
EXPOSE 8082
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
