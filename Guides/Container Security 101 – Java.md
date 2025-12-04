# Container Security 101 – Java  
**Now with real-world Java example**

Secure containers = minimal + hardened + verified + monitored

## Java (Spring Boot / Quarkus / Micronaut) – Secure Dockerfile

```dockerfile
# ================ BUILD ================
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -B

COPY src ./src
RUN ./mvnw package -DskipTests -B

# ================ RUNTIME – TINY & SAFE ================
# ~90–110 MB, zero CVEs, non-root by default
FROM cgr.dev/chainguard/jre:latest

WORKDIR /app
COPY --from=build --chown=65532:65532 /app/target/*.jar app.jar

USER 65532
EXPOSE 8080

ENV JAVA_OPTS="-Xms256m -Xmx512m"

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

### Even smaller? Use Liberica NIK + GraalVM native (~30–60 MB)
```dockerfile
FROM ghcr.io/graalvm/native-image:ol:21 AS build
# ... compile native binary ...

FROM cgr.dev/chainguard/glibc-dynamic:latest
COPY --from=build /app/myapp /app/myapp
USER 65532
ENTRYPOINT ["/app/myapp"]
```

### Distroless alternative (still excellent)
```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
# ... same ...

FROM gcr.io/distroless/java21-debian12
COPY --from=build /app/target/*.jar app.jar
USER nonroot
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

## Final 10-Second Java Checklist

- [ ] Chainguard JRE or Distroless Java base  
- [ ] Multi-stage build  
- [ ] USER 65532  
- [ ] No secrets in JAR/layers  
- [ ] JAR pinned + image signed  
- [ ] Scanned (zero CRITICAL/HIGH)  
- [ ] readOnlyRootFilesystem: true  
- [ ] Capabilities dropped  
- [ ] Memory limits (512Mi typical)  
- [ ] Actuator health endpoint enabled  


