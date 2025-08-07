---
sidebar_position: 2
---

# XADD
The XADD command is the primary method for publishing new messages into a Redis stream. In Redis Streams, producers use XADD to append data entries—commonly referred to as events—to a stream. Each entry consists of a key-value pair and is assigned a unique stream ID.

In this section, we'll start by using XADD via the Redis CLI, and then move on to how it works in a Spring Boot application using Spring Data Redis.


## Redis-CLI
Let’s begin by adding some entries to a stream called __race:france__. Each entry will contain the rider’s name, speed, position, and location ID:
```
$ redis-cli
127.0.0.1:6379> XADD race:france * rider Castilla speed 30.2 position 1 location_id 1
"1754313469817-0"
127.0.0.1:6379> XADD race:france * rider Norem speed 28.8 position 3 location_id 1
"1754313472118-0"
127.0.0.1:6379> XADD race:france * rider Prickett speed 29.7 position 2 location_id 1
"1754313474717-0"
```

Now verify the data with XRANGE:

```
127.0.0.1:6379> XRANGE race:france - +
1) 1) "1754313469817-0"
   2) 1) "rider"
      2) "Castilla"
      3) "speed"
      4) "30.2"
      5) "position"
      6) "1"
      7) "location_id"
      8) "1"
2) 1) "1754313472118-0"
   2) 1) "rider"
      2) "Norem"
      3) "speed"
      4) "28.8"
      5) "position"
      6) "3"
      7) "location_id"
      8) "1"
3) 1) "1754313474717-0"
   2) 1) "rider"
      2) "Prickett"
      3) "speed"
      4) "29.7"
      5) "position"
      6) "2"
      7) "location_id"
      8) "1"
```


:::warning[Important]
In each of the examples, I use `FLUSHDB` to ensure the database starts clean. __Be careful__: if you have an existing Redis instance running on your local machine, `FLUSHDB` will __delete all data in the current database__.

If you're unsure or want to avoid losing important data, the safer approach is to __delete only the specific key__:

```bash
redis-cli DEL race:france
```
:::

To reset the stream before testing from Spring Boot:
```
> flushdb
> XRANGE race:france - +
(empty array)
```

## Spring Boot: XADD with Spring Data Redis

Let’s create a Spring Boot project to send stream entries from an HTTP API:
```
spring init -d=web,data-redis,devtools,thymeleaf,lombok \
  -g com.example \
  -a demo \
  -p jar \
  --build maven \
  xadd -x
```

## Configure Redis
Add to src/main/resources/application.yaml:

```
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

## Define the Application and Controller

DemoApplication.java
```java
@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

}

@Data
class RaceDataRequest {
	private String rider;
	private double speed;
	private int position;
	private int locationId;
}

@RestController
@RequestMapping("/api/race")
@RequiredArgsConstructor
class RaceStreamController {

  private final StringRedisTemplate redisTemplate;

  @PostMapping("/france")
  public String addToRaceFrance(@RequestBody RaceDataRequest request) {
    Map<String, String> fields = Map.of(
        "rider", request.getRider(),
        "speed", String.valueOf(request.getSpeed()),
        "position", String.valueOf(request.getPosition()),
        "location_id", String.valueOf(request.getLocationId())
    );

    redisTemplate.opsForStream().add(MapRecord.create("race:france", fields));
    return "Race data added to stream race:france";
  }
}
```
## Test the Endpoint
Use curl:
```
$ curl -X POST http://localhost:8080/api/race/france -H "Content-Type: application/json" \
  -d {
    "rider": "Castilla",
    "speed": 30.2,
    "position": 1,
    "locationId": 1
}
```

Response:
```
Race data added to stream race:france
```

Verify in Redis:
```
127.0.0.1:6379> XRANGE race:france - +
1) 1) "1754313559708-0"
   2) 1) "location_id"
      2) "1"
      3) "speed"
      4) "30.2"
      5) "position"
      6) "1"
      7) "rider"
      8) "Castilla"
127.0.0.1:6379> 
```
This confirms your Spring Boot API successfully publishes messages into the Redis stream using XADD.

