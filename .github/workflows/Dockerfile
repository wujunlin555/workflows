# Spring PetClinic 简洁 Dockerfile
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app

# 复制 Maven 文件
COPY pom.xml mvnw ./
COPY .mvn .mvn

# 下载依赖
RUN chmod +x mvnw && ./mvnw dependency:go-offline -B

# 复制源代码并构建
COPY src src
RUN ./mvnw clean package -DskipTests -B

# 运行时镜像
FROM eclipse-temurin:17-jre-alpine

# 构建参数
ARG VERSION
ARG GIT_COMMIT
ARG BUILDTIME

# 标签
LABEL app="spring-petclinic" \
      version="${VERSION}" \
      commit="${GIT_COMMIT}"

# 创建用户
RUN adduser -D -s /bin/bash petclinic

# 安装运行时依赖
RUN apk add --no-cache curl tzdata && \
    cp /usr/share/zoneinfo/Asia/Hong_Kong /etc/localtime

# 创建应用目录
WORKDIR /app
RUN chown petclinic:petclinic /app

# 复制 JAR 文件
COPY --from=builder --chown=petclinic:petclinic /app/target/spring-petclinic-*.jar app.jar

# 切换用户
USER petclinic

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# 暴露端口
EXPOSE 8080

# 环境变量
ENV SPRING_PROFILES_ACTIVE=prod \
    JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"

# 启动命令
CMD java $JAVA_OPTS -jar app.jar
