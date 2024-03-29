---
title: "How to make docker build time faster"

categories: 
  - Docker
  - SpringBoot
  - JIB
last_modified_at: now
---
# Having trouble with long build time of your docker image?
Waiting for a long build time is quite difficult to tolerate. <br /><br />
When you are building a docker image, there are some better solutions. <br /><br />
If you are using Dockerfile building an image, please make sure you are using a multi-stage build. <br /><br />
You can make your image leaner, and also take benefit of cached layer of docker. <br /><br />

```
FROM --platform=$BUILDPLATFORM gradle:jdk17-alpine as build
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts ./
COPY gradle gradle/
COPY gradlew ./
COPY src ./src
RUN ./gradlew bootJar -p ./town-app --no-daemon
COPY ./build/libs/app.jar ./
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar", "--spring.profiles.active=local"]
```

- This is a plain Dockerfile. It takes about **140 seconds** in my machine.

![plainTime](/assets/images/plainTime.png)

Now, this is a Dockerfile with multi-stage build.  <br /><br />
Please watch first comment of the script. <br /><br />
Since dependencies of the system rarely changes, it improves build time by caching it’s layer. <br /><br />
![multistageTime](/assets/images/multistageTime.png)
- **130 seconds** without that command, <br/>

![cachedTime](/assets/images/cachedTime.png)

- About **86 seconds** with that command.

```
FROM --platform=$BUILDPLATFORM gradle:jdk17-alpine as build
WORKDIR /app

COPY build.gradle.kts settings.gradle.kts ./
COPY gradle gradle/
COPY gradlew ./

RUN gradle --no-daemon build || return 0  # Build for cache dependencies
COPY src ./src
RUN ./gradlew bootJar -p ./app --no-daemon


FROM --platform=$TARGETPLATFORM eclipse-temurin:17-jre-alpine
WORKDIR /usr/local

COPY --from=build /app/build/libs/app.jar ./

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "-Duser.timezone=Asia/Seoul", "-Xmx2048m", "app.jar", "--spring.profiles.active=local"]
```

Lastly, If you are using gradle or maven, consider using JIB for building docker image. <br /><br />
It is noted for jib from the google document that, <br /><br />
**“It reads your build config, organizes your application into distinct layers (dependencies, resources, classes) and only rebuilds and pushes the layers that have changed.”** <br /><br />
Jib provides much faster build time than using Dockerfile. <br /><br />
![jibTime](/assets/images/jibTime.png) <br /><br />
It took about 6 seconds to build. <br /><br />
The image size is almost the same, 241.64MB for using Dockerfile and 241.55MB for using JIB. <br /><br />

```
jib {
    val activeProfile = if (project.hasProperty("profile")) project.property("profile") else "local"
    from {
        image = "eclipse-temurin:17-jre-alpine"
    }
    to {
        image = “app”
        tags = setOf("latest")
    }
    container {
        mainClass = "silver.town.TownApplication"
        creationTime = "USE_CURRENT_TIMESTAMP"
        ports = listOf("8080")
        jvmFlags = listOf("-Dspring.profiles.active=$activeProfile", "-XX:InitialRAMPercentage=75", "-XX:MinRAMPercentage=75", "-XX:MaxRAMPercentage=75", "-XX:+UseContainerSupport")
    }
}
```

If you want to test it in your local system, use *./gradlew jibDockerBuild* to only push to your local docker desktop. <br /><br />

### Conclusion
- If you are using gradle or maven, consider using JIB for building your docker image.
- If you want to use Dockerfile, use multi-stage build, try to cache as much as possible for faster build time.

This article was created by Crocoder7. It is not to be copied without permission.

#### References
  * [Docker Document](https://www.docker.com/blog/9-tips-for-containerizing-your-spring-boot-code/)
  * [Google Document](https://cloud.google.com/blog/products/application-development/introducing-jib-build-java-docker-images-better?hl=en)
