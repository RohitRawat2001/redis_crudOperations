# Redis
Redis can be used as a database, cache, streaming engine, message broker, and more. The following quick start guides will show you how to use Redis for the following specific purposes:

# Data structure store
```json
What is it? Redis can store different types of simple data.
How it works:
You can store text, lists, sets (groups of unique items), and more.
It’s great for quickly saving and retrieving data, like a chat history or a leaderboard in a game.
```
# Document database
```json
What is it? Redis can store data in a flexible format, like a file or document.
How it works:
You can store data that looks like a JSON file, where each user or item can have different fields (like a user’s name, age, or hobbies).
It’s useful when you don’t know exactly what data you’ll have ahead of time
```
# Vector database 
```json
What is it? Redis can store special data called vectors (groups of numbers) used in AI.
How it works:
Vectors are used in things like image recognition or product recommendations.
Redis can store these vectors and quickly find similar ones, like finding items you might like based on past choices.
```

# redis_crudOperations
Step-by-Step Redis
# 1. User Data Transfer Object (UserDto)
Code:
```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@ToString
public class UserDto implements Serializable {
    
    @Id
    private String userId;
    private String name;
    private String email;
    private String password;
    private String about;
}
```

# Explanation:

```json
- UserDto is a simple data structure used to transfer user-related data.
-- Annotations:
- @Getter and @Setter: Generate getter and setter methods automatically.
- @NoArgsConstructor and @AllArgsConstructor: Automatically create constructors (no-args and all-args).
- @ToString: Generates a toString() method for the object.
- @Id: Marks userId as the identifier.
- Serializable: Allows the object to be converted into a byte stream so it can be stored in Redis (which uses serialized data).
```

# 2. Redis Repository (UserDao)

Code:
```java
@Repository
public class UserDao {

    private static final String HASH_KEY = "USER";

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public UserDto save(UserDto user) {
        redisTemplate.opsForHash().put(HASH_KEY, user.getUserId(), user);
        return user;
    }

    public UserDto get(String userId) {
        return (UserDto) redisTemplate.opsForHash().get(HASH_KEY, userId);
    }

    public Map<Object, Object> findAll() {
        return redisTemplate.opsForHash().entries(HASH_KEY);
    }

    public UserDto updateUser(String userId, UserDto updatedUser) {
        UserDto existingUser = get(userId);
        if (existingUser == null) {
            return null;
        }
        existingUser.setName(updatedUser.getName());
        existingUser.setEmail(updatedUser.getEmail());
        existingUser.setPassword(updatedUser.getPassword());
        existingUser.setAbout(updatedUser.getAbout());
        save(existingUser);
        return existingUser;
    }

    public void deleteUser(String userId) {
        redisTemplate.opsForHash().delete(HASH_KEY, userId);
    }
}
```
# Explanation:
```json
- Repository Pattern: This class handles interactions with Redis.
- HASH_KEY: Redis stores data as key-value pairs; HASH_KEY ("USER") groups all users in a hash map.
CRUD Methods:
- save: Saves a user object into Redis, using the userId as the key.
- get: Retrieves a specific user from Redis by userId.
- findAll: Retrieves all users stored in the USER hash from Redis.
- updateUser: Updates an existing user’s details in Redis and returns the updated user.
- deleteUser: Deletes a user from Redis by userId.
RedisTemplate: Spring Boot component used to communicate with Redis. It provides helper methods to perform operations like saving or retrieving data from Redis hashes.
```
# 3. Redis Configuration (RedisConfig)
Code:

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory();
    }

    @Bean
    public RedisSerializer<Object> redisSerializer() {
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
        return serializer;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(connectionFactory());
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return redisTemplate;
    }
}
```
# Explanation:
```json
- Configuration Class: Sets up how the application will connect to Redis.
- RedisConnectionFactory: Creates a factory for managing Redis connections, here using Lettuce (a Redis client).
RedisTemplate:
- The template that allows interactions with Redis, including serialization of keys and values.
- Key Serializer: StringRedisSerializer ensures the key (userId) is stored as a string.
- Value Serializer: GenericJackson2JsonRedisSerializer serializes UserDto objects into JSON format so they can be stored in Redis.
```

# 4. User Controller (UserDtoController)
Code:

```java
@RestController
@RequestMapping("/api/userdto")
public class UserDtoController {

    @Autowired
    private UserDao userDao;

    private Logger logger = LoggerFactory.getLogger(UserDtoController.class);

    @PostMapping("/")
    public ResponseEntity<UserDto> createUser(@RequestBody UserDto user) {
        String userId = UUID.randomUUID().toString();
        user.setUserId(userId);
        UserDto createdUser = userDao.save(user);
        return new ResponseEntity<>(createdUser, HttpStatus.CREATED);
    }

    @GetMapping("/all")
    public ResponseEntity<Map<Object, Object>> getAllUsers() {
        Map<Object, Object> allUsers = userDao.findAll();
        return new ResponseEntity<>(allUsers, HttpStatus.OK);
    }

    @GetMapping("/{userId}")
    public ResponseEntity<UserDto> getUser(@PathVariable String userId) {
        UserDto user = userDao.get(userId);
        return new ResponseEntity<>(user, HttpStatus.OK);
    }

    @PutMapping("/{userId}")
    public ResponseEntity<UserDto> updateUser(@RequestBody UserDto updatedUser, @PathVariable String userId) {
        UserDto updated = userDao.updateUser(userId, updatedUser);
        return new ResponseEntity<>(updated, HttpStatus.OK);
    }

    @DeleteMapping("/{userId}")
    public ResponseEntity<ApiResponse> deleteUser(@PathVariable String userId) {
        userDao.deleteUser(userId);
        return new ResponseEntity<>(new ApiResponse("User deleted successfully", true, HttpStatus.OK), HttpStatus.OK);
    }
}
```
# Explanation:
```json
- REST Controller: Provides endpoints for interacting with user data stored in Redis.
Endpoints:
- Create User (POST /api/userdto/):
Generates a random UUID for the new user and saves the user to Redis.
Returns the created user with status 201 Created.
- Get All Users (GET /api/userdto/all):
Retrieves all users stored in Redis and returns them with status 200 OK.
- Get User by ID (GET /api/userdto/{userId}):
Fetches a single user from Redis based on their userId and returns it with status 200 OK.
- Update User (PUT /api/userdto/{userId}):
Updates the specified user’s details in Redis and returns the updated user with status 200 OK.
- Delete User (DELETE /api/userdto/{userId}):
Deletes the user with the given userId and returns a success response with status 200 OK.
```
5. Spring Boot Configuration for Redis
Code (application.properties):
```java
spring.redis.host=localhost
spring.redis.port=6379
```

# Explanation:
```json
Configures Redis to run on localhost (local machine) and on port 6379 (the default Redis port).
```
# 6. Maven Dependency for Redis
Code (pom.xml):
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
# Explanation:
```
Adds the necessary dependency for Spring Boot to interact with Redis using the RedisTemplate component. This pulls in all the required libraries for Redis integration.
```

