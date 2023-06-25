---
title: "ElastiCache with keyspace event"

categories: 
  - ElastiCache
  - Redis
  - Spring
last_modified_at: now
---
# Problems when using ElastiCache with keyspace event
faced this error when I tried to use notify-keyspace-events option in spring data redis.
```
Caused by: org.springframework.data.redis.RedisSystemException: Error in execution;
nested exception is io.lettuce.core.RedisCommandExecutionException: ERR unknown command `CONFIG`, with args beginning with: `GET`, `notify-keyspace-events`
```
- In my case, I was using AWS Elasticache which does not allow CONFIG command.
- I was able to solve this error by setting empty string in keyspaceNotificationsConfigParameter
```
@EnableRedisRepositories(
    enableKeyspaceEvents = RedisKeyValueAdapter.EnableKeyspaceEvents.ON_STARTUP,
    keyspaceNotificationsConfigParameter = ""
)
```
- By using this option, I was able to stop sending CONFIG command to Elasticache.
- It will keep the existing Elasticache configuration.
- The default value is 'Ex'
- Since the configuration of Elasticache cannot be changed by CONFIG command, notify-keyspace-events option should be manually changed in AWS before using it.
- [AWS Guide](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/ParameterGroups.Redis.html#ParameterGroups.Redis.2-8-6) will help configuring notify-keyspace-events option.
- The option should be 'Ex'
