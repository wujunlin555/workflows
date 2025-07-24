# 多阶段构建的 Java Spring Boot Dockerfile
# 兼容 Maven 和 Gradle 项目，支持 Java 8/11/17/21

# ====================================
# 阶段 1: 构建阶段
# ====================================
FROM eclipse-temurin:17-jdk-alpine AS builder

# 设置构建参数（从 GitHub Actions 传入）
ARG BUILDTIME
ARG VERSION
ARG GIT_COMMIT
ARG GIT_BRANCH

# 设置工作目录
WORKDIR /app

# 安装必需的构建工具
RUN apk add --no-cache \
    curl \
    bash \
    findutils

# 复制构建文件
COPY pom.xml* build.gradle* gradle.properties* gradlew* settings.gradle* ./
COPY gradle/ gradle/ 2>/dev/null || true

# 根据项目类型下载依赖
RUN if [ -f "pom.xml" ]; then \
        echo "📦 检测到 Maven 项目，下载依赖..."; \
        # 使用 Maven Wrapper 或系统 Maven
        if [ -f "mvnw" ]; then \
            chmod +x mvnw && ./mvnw dependency:go-offline -B; \
        else \
            # 安装 Maven
            apk add --no-cache maven && \
            mvn dependency:go-offline -B; \
        fi; \
    elif [ -f "build.gradle" ] || [ -f "build.gradle.kts" ]; then \
        echo "📦 检测到 Gradle 项目，下载依赖..."; \
        chmod +x gradlew && \
        ./gradlew dependencies --no-daemon; \
    else \
        echo "⚠️ 未检测到构建文件，跳过依赖下载"; \
    fi

# 复制源代码
COPY src/ src/

# 构建应用
RUN if [ -f "pom.xml" ]; then \
        echo "🏗️ 使用 Maven 构建..."; \
        if [ -f "mvnw" ]; then \
            ./mvnw clean package -DskipTests -B; \
        else \
            mvn clean package -DskipTests -B; \
        fi; \
        # 找到构建的 JAR 文件
        find target -name "*.jar" -not -name "*-sources.jar" -not -name "*-javadoc.jar" -exec cp {} app.jar \;; \
    elif [ -f "build.gradle" ] || [ -f "build.gradle.kts" ]; then \
        echo "🏗️ 使用 Gradle 构建..."; \
        ./gradlew build -x test --no-daemon; \
        # 找到构建的 JAR 文件
        find build/libs -name "*.jar" -not -name "*-plain.jar" -exec cp {} app.jar \;; \
    else \
        echo "❌ 未找到构建文件，构建失败"; \
        exit 1; \
    fi

# 验证 JAR 文件是否存在
RUN if [ ! -f "app.jar" ]; then \
        echo "❌ 构建失败：找不到 JAR 文件"; \
        ls -la target/ build/libs/ 2>/dev/null || true; \
        exit 1; \
    else \
        echo "✅ 构建成功，JAR 文件大小: $(du -h app.jar | cut -f1)"; \
    fi

# ====================================
# 阶段 2: 运行时阶段
# ====================================
FROM eclipse-temurin:17-jre-alpine AS runtime

# 设置构建参数
ARG BUILDTIME
ARG VERSION
ARG GIT_COMMIT
ARG GIT_BRANCH

# 设置标签
LABEL maintainer="your-team@company.com" \
      org.opencontainers.image.title="Java Spring Boot Application" \
      org.opencontainers.image.description="Production-ready Java application" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.created="${BUILDTIME}" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.source="https://github.com/your-org/your-repo" \
      build.time="${BUILDTIME}" \
      build.version="${VERSION}" \
      build.commit="${GIT_COMMIT}" \
      build.branch="${GIT_BRANCH}"

# 创建非 root 用户
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# 安装运行时依赖
RUN apk add --no-cache \
    ca-certificates \
    tzdata \
    curl \
    bash \
    dumb-init && \
    # 设置时区
    cp /usr/share/zoneinfo/Asia/Hong_Kong /etc/localtime && \
    echo "Asia/Hong_Kong" > /etc/timezone

# 创建应用目录
RUN mkdir -p /app/config /app/logs /app/tmp && \
    chown -R appuser:appgroup /app

# 设置工作目录
WORKDIR /app

# 从构建阶段复制 JAR 文件
COPY --from=builder --chown=appuser:appgroup /app/app.jar ./app.jar

# 创建启动脚本
RUN cat > start.sh << 'EOF'
#!/bin/bash
set -e

# 打印启动信息
echo "=================================="
echo "🚀 Starting Java Application"
echo "📦 Version: ${VERSION:-unknown}"
echo "🔗 Commit: ${GIT_COMMIT:-unknown}"
echo "🌿 Branch: ${GIT_BRANCH:-unknown}"
echo "⏰ Build Time: ${BUILDTIME:-unknown}"
echo "🕒 Current Time: $(date)"
echo "💾 Available Memory: $(free -h | grep Mem | awk '{print $7}')"
echo "🖥️  CPU Cores: $(nproc)"
echo "=================================="

# JVM 参数设置
JAVA_OPTS="${JAVA_OPTS} -XX:+UseContainerSupport"
JAVA_OPTS="${JAVA_OPTS} -XX:MaxRAMPercentage=75.0"
JAVA_OPTS="${JAVA_OPTS} -XX:+UseG1GC"
JAVA_OPTS="${JAVA_OPTS} -XX:+UseStringDeduplication"
JAVA_OPTS="${JAVA_OPTS} -XX:+PrintGCDetails"
JAVA_OPTS="${JAVA_OPTS} -XX:+PrintGCTimeStamps"
JAVA_OPTS="${JAVA_OPTS} -Xloggc:/app/logs/gc.log"
JAVA_OPTS="${JAVA_OPTS} -XX:+UseGCLogFileRotation"
JAVA_OPTS="${JAVA_OPTS} -XX:NumberOfGCLogFiles=5"
JAVA_OPTS="${JAVA_OPTS} -XX:GCLogFileSize=10M"

# Spring Boot 参数
JAVA_OPTS="${JAVA_OPTS} -Dspring.profiles.active=${SPRING_PROFILES_ACTIVE:-prod}"
JAVA_OPTS="${JAVA_OPTS} -Dserver.port=${SERVER_PORT:-8080}"
JAVA_OPTS="${JAVA_OPTS} -Djava.security.egd=file:/dev/./urandom"
JAVA_OPTS="${JAVA_OPTS} -Dfile.encoding=UTF-8"
JAVA_OPTS="${JAVA_OPTS} -Duser.timezone=Asia/Hong_Kong"

# 健康检查参数
JAVA_OPTS="${JAVA_OPTS} -Dmanagement.endpoints.web.exposure.include=health,info,metrics,prometheus"
JAVA_OPTS="${JAVA_OPTS} -Dmanagement.endpoint.health.show-details=always"

# 如果设置了调试模式
if [ "${JAVA_DEBUG}" = "true" ]; then
    echo "🐛 Debug mode enabled"
    JAVA_OPTS="${JAVA_OPTS} -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"
fi

# 打印最终的 JVM 参数
echo "🔧 JVM Options: ${JAVA_OPTS}"
echo "=================================="

# 启动应用
exec java ${JAVA_OPTS} -jar app.jar "$@"
EOF

# 设置启动脚本权限
RUN chmod +x start.sh && chown appuser:appgroup start.sh

# 切换到非 root 用户
USER appuser

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:${SERVER_PORT:-8080}/actuator/health || exit 1

# 暴露端口
EXPOSE 8080 5005

# 设置环境变量
ENV SPRING_PROFILES_ACTIVE=prod \
    SERVER_PORT=8080 \
    JAVA_OPTS="" \
    TZ=Asia/Hong_Kong

# 使用 dumb-init 作为 PID 1，避免僵尸进程
ENTRYPOINT ["dumb-init", "--"]

# 启动命令
CMD ["./start.sh"]

# ====================================
# 构建时验证
# ====================================
# 在构建时运行一些基本检查
RUN echo "✅ Dockerfile 构建完成" && \
    echo "📁 工作目录内容:" && \
    ls -la && \
    echo "📦 JAR 文件信息:" && \
    file app.jar && \
    echo "👤 当前用户: $(whoami)" && \
    echo "🏠 Home 目录: $HOME"
