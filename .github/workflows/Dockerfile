# 简化的 Spring PetClinic Dockerfile
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app

# 复制构建文件
COPY pom.xml mvnw ./
COPY .mvn .mvn

# 下载依赖并构建
RUN chmod +x mvnw && \
    ./mvnw dependency:go-offline -B

COPY src src
RUN ./mvnw clean package -DskipTests -B

# 运行时镜像
FROM eclipse-temurin:17-jre-alpine

# 构建参数
ARG VERSION
ARG GIT_COMMIT

# 创建用户和目录
RUN adduser -D petclinic && \
    apk add --no-cache curl

WORKDIR /app
USER petclinic

# 复制应用
COPY --from=builder --chown=petclinic:petclinic /app/target/spring-petclinic-*.jar app.jar

# 健康检查
HEALTHCHECK CMD curl -f http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

# 启动
CMD ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
