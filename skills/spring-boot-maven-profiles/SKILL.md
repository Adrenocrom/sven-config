---
name: spring_boot_maven_profiles
description: How to map Spring Boot profiles to Maven profiles for activation via
  mvn spring-boot:run -P<profile>
tags:
- spring-boot
- maven
- profiles
- configuration
- spring-boot-maven-plugin
created_at: '2026-07-16T13:56:22.604390+00:00'
---

To configure Spring Boot profiles that can be activated via Maven command-line arguments (e.g. `mvn spring-boot:run -Pllmintern`):

1. Create profile-specific property files in `src/main/resources`:
   - `application.properties` — default/shared config
   - `application-llmintern.properties` — config for the `llmintern` Spring profile
   - `application-prod.properties` — example second profile

2. In `pom.xml`, define Maven profiles that set the `spring-boot.run.profiles` property recognized by the Spring Boot Maven plugin:

```xml
<profiles>
    <profile>
        <id>llmintern</id>
        <properties>
            <spring-boot.run.profiles>llmintern</spring-boot.run.profiles>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <spring-boot.run.profiles>prod</spring-boot.run.profiles>
        </properties>
    </profile>
</profiles>
```

3. Run with: `mvn spring-boot:run -Pllmintern`

4. Notes:
   - `-P` activates Maven profiles; this mapping bridges them to Spring profiles.
   - `spring-boot.run.profiles` is consumed by the `spring-boot-maven-plugin` during `spring-boot:run`.
   - This does not affect the packaged JAR. At runtime use: `java -jar target/myapp.jar --spring.profiles.active=llmintern`
   - To make the active profile visible in the app, inject `@Value("${spring.profiles.active}")` or log environment properties.
