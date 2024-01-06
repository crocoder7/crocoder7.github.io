---
title: "Design of chatting system with Websocket, RDB, and Redis"

categories: 
  - Spring Boot
  - System Design
  - Websocket
  - Redis
last_modified_at: now
---
# How I designed chatting system with Websocket, RDB, and Redis 
Objectives : 
  1. The chatting message should be sent through Websocket STOMP and saved to the database for later read. 
  2. Users can find out unread count of the chatting message that other users sent in the chat room.
---
- This is how I designed chatting system with RDB and Redis. 
- Since I used Spring Boot for backend, the detail implementation could be modified for your case.

### First, STOMP can be easily implemented in Spring Boot.

```
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

	@Override
	public void registerStompEndpoints(StompEndpointRegistry registry) {
		registry.addEndpoint(“/websocket”).setAllowedOriginPatterns(“*”).withSockJS();
	}

	@Override
	public void configureMessageBroker(MessageBrokerRegistry registry) {
		registry.enableSimpleBroker(“/sub”);
	}
}

@Controller
public class STOMPController {

	@MessageMapping(“/pub/1”)
	public void handle(String message) {
		// saving message to database and so on..
		// increase unread count of offline users only
	}
}

@RestController
public class ChatController {

	@PutMapping(“/chat/clear-unread“)
	public void clearUnreadCount() {
		// set unread count to zero
		// only the first inquiry of chat messages
	}
}
```

- Under the MessageMapping annotation, saving message could be implemented. 
- Unread message count of the user should be incremented only when user is offline so that we can save extra network communication. 

![beforeChat](/assets/images/beforeChat.png)

- If unread count of the message incremented, regardless of the user’s online status, the API that sets unread count to zero should be automatically called every time when other users in the chat room send message for online user. 

![chatErd](/assets/images/chatErd.png)

- This is the table schema of RDS.
- Incrementing and resetting the unread count could be the cause of the late response under multiple requests, since it would update the same row at the same time.

### How to keep track of online users in the chat room.
- STOMP create session for each websocket creation. 
- We need to create redis session based on both websocket session and user id to distinguish multiple connection by one user. (If one user uses both phone and laptop to use application, there will be two different websocket session created for that one user.)

- So, we need three redis session. 
  1. for the user websocket session. 
  2. for the chat room subscription. 
  3. for the chat room.

Chat room subscription session is needed since STOMP create its’s session for each subscription for channel, not the chat room id.  
  
- Let’s say if you subscribe to the path “/sub/chat/1”.
- The STOMP would create session something like “sub-1010” and it would be used when disconnected.
- Therefore, we need this subscription session for user to leave the chat room.
- If the chat room id is contained in the path of the subscription, we can easily extract that and use it to make chat room session.

The chat room session contains chat room id and user STOMP sessions followed by user id. In that way, we can find the live users in the chat room. 

```
public class ChatRoom {
	private String id;
	private Map<String, String> sessionKeyUserIdValue;
}
```

- Creating and deleting session could be implemented by implementing ChannelInterceptor class.

```
public class CustomChannelInterceptor implements ChannelInterceptor {

	@Override
	public Message<?> preSend(Message<?> message, MessageChannel channel) {
		
		StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
		StompCommand command = accessor.getStompCommand();
		String sessionId = accessor.getSessionId();
		String destination = accessor.getDestination();  // needed for subscribe
		String subscriptionId = accessor.getSubscriptionId();
		switch (accessor.getCommand()) {
			case CONNECT -> // user validation could be implemented
			case SUBSCRIBE -> // joining chat room need to be implemented
			case UNSUBSCRIBE -> // leaving chat room need to be implemented. Deleting subscription session
			case DISCONNECT ->  // leaving chat room and deleting user session need to be implemented
		}
		return message;
	}
}
```

For both, unsubsription and disconnection command, we can remove the live user from the chat room session.  
- Since redis was used for session storage, lua script was used when joining the chat room and leaving the chat room to ensure the atomicity.
- The chat room session could be overwrited if two chat room creation is requested at the same time.
- Please make sure that redis does NOT ensure atomicity if your command is a “read and modify cycle”.
- You should use either multi/exec or lua script for atomicity.  
- The script would be something like below

```
local isRoomExist = redis.call(‘EXISTS’, KEYS[1])
if isRoomExist == 1 then
	local userSessionIds = redis.call('HGET', KEYS[1], 'sessionKeyUserIdValue')
	local joinUserSessionId = ARGV[1]
	local joinUserId = ARGV[2]
	local newSessionIds = cjson.decode(userSessionIds)
	newSessionIds[joinUserSessionId] = joinUserId

	redis.call('HSET', KEYS[1], 'sessionKeyUserIdValue', cjson.encode(newSessionIds))
	redis.call('EXPIRE', KEYS[1], KEYS[2])
else
	local joinUserSessionId = ARGV[1]
	local joinUserId = ARGV[2]
	redis.call('HSET', KEYS[1], 'sessionKeyUserIdValue', cjson.encode({[joinUserSessionId] = joinUserId}))
	redis.call('EXPIRE', KEYS[1], KEYS[2])
end
```

- Where KEYS[1] is chat room session id and KEYS[2] is TTL.
- Implementing leaving the chat room functionality can be done in a similar manner.
- Now, we are ready to use live chat room user list!
- It can be used to increment the unread count and also it can be further implemented for live user status of the chat room on the UI.  

### Lastly, since we can also use redis for message broker. 
To make the system scalable, external message broker is necessary. 

```
@Configuration
public class RedisConfiguration {
	private final CUSTOM_PATTERN_TOPIC = new PatternTopic(“/sub/*”);
	
	@Bean
	public RedisMessageListenerContainer redisMessageListenerContainer(
		RedisConnectionFactory connectionFactory,
		CustomMessageListener customMessageListener
	) {
		RedisMessageListenerContainer container = new RedisMessageListenerContainer();
		container.addMessageListener(customMessageListener, CUSTOM_PATTERN_TOPIC);
		return container;
	}
}

@Service
public class CustomMessageListener extends MessageListener {
	private final SimpleMessageSendingOperations simpleMessageSendingOperations;

	@Override
	void onMessage(Message message, byte[] pattern) {
		// implement payload and destination topic to send user
		simpleMessageSendingOperations.convertAndSend(destination, payload);
	}
}
```

- Configuration and message subscription part can be implemented like above. 
- Publishing to redis message broker can be done by simply sending message to certain topic.(See below)  

```
redisTemplate.convertAndSend(topic, payload);
```

- Now, the system is scalable and reasonable enough to me.
- The overview of the design of the system would be like below.

![chatDesign](/assets/images/finalChat.png)

* References
  * [Spring Doc](https://docs.spring.io/spring-framework/reference/web/websocket/stomp/message-flow.html)
  * [Spring Doc](https://docs.spring.io/spring-framework/reference/web/websocket/stomp/interceptors.html)
