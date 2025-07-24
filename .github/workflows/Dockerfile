FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app

# 复制 Maven Wrapper 和配置文件
COPY mvnw pom.xml ./
COPY .mvn .mvn

# 给 mvnw 执行权限并下载依赖
RUN chmod +x mvnw && ./mvnw dependency:go-offline -B

# 复制源代码并构建
COPY src src
RUN ./mvnw clean package -DskipTests -B

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
