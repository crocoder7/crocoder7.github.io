---
title: "Prometheus JVM Spring metric"

categories: 
  - Prometheus
  - JVM
  - Spring
  - DevOps
last_modified_at: now
---
# How to get centralize JVM & Spring metric using Prometheus
Amazon managed prometheus and grafana might be a good option for monitoring when you are AWS EKS. <br>

```
java.lang.IllegalAccessError: *** cannot access class com.sun.crypto.provider.SunJCE (in module java.base) because module java.base does not export com.sun.crypto.provider to unnamed module
```
This error could be solved by copying java.security when making image using dockerfile.
```
COPY --from=build /opt/java/openjdk/conf/security/java.security
```
Adding copy command from the build stage in dockerfile, which was using jdk11 or 17 as base image, resolved the SunJCE access error. <br>
When using jib instead of dockerfile requires different solution than previous, since that kind of copying command is impossible in JIB. <br><br>
Here are the solution when using JIB. <br>
Adding jvm option has solved the problem
```
jib {
  ...
  container {
    jvmFlags = ["--add-exports=java.base/com.sun.crypto.provider=ALL-UNNAMED"]
  }
}
```

* References
  * [Blog](https://medium.com/@damindubandara/configuring-jvm-monitoring-dashboard-in-grafana-using-spring-boot-actuator-with-prometheus-and-e7142b5e4c81)
  * [Blog](https://malwareanalysis.tistory.com/602)
