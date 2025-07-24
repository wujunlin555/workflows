FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app

COPY . ./

RUN ./mvnw clean package -DskipTests 

FROM eclipse-temurin:17-jre-alpine

ARG VERSION
ARG GIT_COMMIT

RUN adduser -D petclinic && apk add --no-cache curl

WORKDIR /app
USER petclinic

COPY --from=builder --chown=petclinic:petclinic /app/target/spring-petclinic-*.jar app.jar

HEALTHCHECK CMD curl -f http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

CMD ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
