# module-queue/Dockerfile
# Build stage
FROM gradle:8.5-jdk21 AS builder

WORKDIR /app

# Gradle 설정 파일들 복사
COPY gradle gradle
COPY gradlew .
COPY settings.gradle .
COPY build.gradle .
RUN chmod +x gradlew

# 모든 모듈의 build.gradle 복사 (settings.gradle에 정의된 모든 모듈)
COPY module-common/build.gradle module-common/
COPY module-core/build.gradle module-core/
COPY module-auth/build.gradle module-auth/
COPY module-metric/build.gradle module-metric/
COPY module-api/build.gradle module-api/
COPY module-queue/build.gradle module-queue/
COPY module-api-admin/build.gradle module-api-admin/
COPY authorization-spring-boot-starter/build.gradle authorization-spring-boot-starter/

# 모든 모듈 소스 코드 복사
COPY module-common module-common
COPY module-core module-core
COPY module-auth module-auth
COPY module-metric module-metric
COPY module-api module-api
COPY module-queue module-queue
COPY module-api-admin module-api-admin
COPY authorization-spring-boot-starter authorization-spring-boot-starter

# 의존성 다운로드
RUN ./gradlew :module-queue:dependencies --no-daemon || true

# Queue 모듈만 빌드
RUN ./gradlew :module-queue:bootJar --no-daemon --stacktrace

# Runtime stage
FROM eclipse-temurin:21-jre

WORKDIR /app

# curl 설치
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*

# JAR 파일 복사
COPY --from=builder /app/module-queue/build/libs/*.jar app.jar

EXPOSE 8083

# 헬스체크
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8083/actuator/health || exit 1

# 실행
ENTRYPOINT ["java", \
    "-Xms256m", \
    "-Xmx512m", \
    "-XX:+UseG1GC", \
    "-XX:+UseContainerSupport", \
    "-Dspring.profiles.active=prod", \
    "-jar", "/app/app.jar"]