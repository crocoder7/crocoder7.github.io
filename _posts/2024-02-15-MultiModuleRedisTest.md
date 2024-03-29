---
title: "Test code for SpringBoot multi-module with Redis"

categories: 
  - SpringBoot
  - Test
  - Redis
  - Multi-module
  - gradle
last_modified_at: now
---
# Test code environment in SpringBoot multi-module project with Redis utilized in a child module.

In multi-module system, setting up environment for integration test code can be pain in the ass.<br/><br/>
Especially, when the child module uses database connection. <br/><br/>
For testing environment, temporary database should be up and running for (relatively)short period of time and down after when test is done. <br/><br/>
Embedded redis can be an option if your child module uses redis.<br/><br/><br/>

In gradle, testFixture can be used for parent module to exploit configuration of child module. <br/><br/>
**Let’s say that parent module is called ‘A’ and child module is called ‘B’.** <br/><br/>
As shown below, testFixture should be set up in build.gradle file of module B. <br/><br/>

```
plugins {
    id "java-test-fixtures"
}

dependencies {
  testFixturesImplementation 'org.springframework.boot:spring-boot-starter-data-redis'
  testFixturesImplementation group: 'it.ozimov', name: 'embedded-redis', version: '0.7.2'
  testFixturesImplementation ('org.springframework.boot:spring-boot-starter-web') // include if your module does not use it on implementation
}
```

<br/>
Now, it should be included in build.gradle file of module A. 
<br/><br/>

```
dependencies {
  testImplementation(testFixtures(project(':B')))
}
```

<br/>
After setting up for build.gradle, it's time to write code for embedded redis up and running. <br/><br/>
Following two classes should be configured under testFixtures path in module B. <br/><br/>

```
@Import(TestRedisConfig.class)
@Configuration("redisTest")
public class EmbeddedRedis {
    @Value("${spring.redis.port}")
    private int port;
    private RedisServer redisServer;

    @PostConstruct
    public void startRedis() {
        this.redisServer = new RedisServer(port);
        this.redisServer.start();
    }

    @PreDestroy
    public void stopRedis() {
        if (redisServer != null) {
            this.redisServer.stop();
        }
    }
}
```

```
@DependsOn("redisTest")
@Configuration
public class TestRedisConfig {

    @Value("${spring.redis.host}")
    private String host;
    @Value("${spring.redis.port}")
    private int port;

    @Primary
    @Bean
    public RedisConnectionFactory testLettuceConnectionFactory() {
        return new LettuceConnectionFactory(host, port);
    }

    @Primary
    @Bean
    public StringRedisTemplate testStringRedisTemplate(RedisConnectionFactory redisConnectionFactory) {
        StringRedisTemplate stringRedisTemplate = new StringRedisTemplate();
        stringRedisTemplate.setConnectionFactory(redisConnectionFactory);

        return stringRedisTemplate;
    }
}
```
In TestRedisConfig class, DependsOn annotation is used to prevent LettuceConnectionFactory creation before the embedded redis running. <br/><br/>
The primary annotation is added to prevent multiple bean error, since the configuration for actual server also exist in main directory. <br/><br/>
Please make sure that you have configured **spring.redis.hot and port** properly in your yml or properties file under testFixtures/resources directory. <br/><br/><br/>

Module A should import the configuration above.

```
@SpringBootTest(
  classes = {EmbeddedRedis.class},
  properties = "spring.profiles.active:b-test"
)
public class ParentTest {
}
```

Note that properties option configured properly with the name of yml file in module B. <br/><br/>
In this case, **application-b-test.yml** should be the name of the yml file in module B. <br/><br/>

This article was created by Crocoder7. It is not to be copied without permission.

#### References
  * [Gradle Document](https://docs.gradle.org/current/userguide/java_testing.html#consuming_test_fixtures_of_another_project)
