---
title: "Cannot access SunJCE error in JDK11 above"

categories: 
  - Java
  - JDK17
last_modified_at: now
---
# SunJCE error when using it in JDK 11 or above. How to solve it when using Jib
I faced error below when I migrated from dockerfile to jib.
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
  * [Stackoverflow](https://stackoverflow.com/questions/61957114/how-to-resolve-the-type-com-sun-crypto-provider-sunjce-is-not-accessible-with)
