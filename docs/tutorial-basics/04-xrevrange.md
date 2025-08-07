---
sidebar_position: 4
---

#  XREVRANGE
In some use cases, you may need to read stream entries in **reverse order**, from the **latest** to the **earliest**. 

While Redis Streams do not support traditional sorting mechanisms like in relational databases, you can traverse the stream in reverse using the `XREVRANGE` command.

## Basic Syntax
```bash
XREVRANGE [key] [end] [start] COUNT [count]
```
***:bulb: Unlike `XRANGE`, the start and end positions are reversed.***


## Example: Read Entries 50 → 46
Let’s revisit our previously generated stream (race:france) and read the 5 most recent entries (starting from position 50):

```
XREVRANGE race:france + - COUNT 5
1) 1) "1754326272465-49"
   2) 1) "rider"
      2) "Roglic"
      3) "speed"
      4) "20.3"
      5) "position"
      6) "50"
      7) "location_id"
      8) "3"
2) 1) "1754326272465-48"
   2) 1) "rider"
      2) "Fernandez"
      3) "speed"
      4) "22.5"
      5) "position"
      6) "49"
      7) "location_id"
      8) "5"
3) 1) "1754326272465-47"
   2) 1) "rider"
      2) "Lopez"
      3) "speed"
      4) "20.8"
      5) "position"
      6) "48"
      7) "location_id"
      8) "4"
4) 1) "1754326272465-46"
   2) 1) "rider"
      2) "Moreira"
      3) "speed"
      4) "37.3"
      5) "position"
      6) "47"
      7) "location_id"
      8) "4"
5) 1) "1754326272465-45"    <--- Ends here (position 46)
   2) 1) "rider"
      2) "Van Aert"
      3) "speed"
      4) "27.1"
      5) "position"
      6) "46"              
      7) "location_id"
      8) "5"
```

## Page 2: Read Entries 45 → 41
To continue paging backwards, decrement the sequence portion of the last ID (1754326272465-45) by one:

```
127.0.0.1:6379> xrevrange race:france 1754326272465-44 - count 5

1) 1) "1754326272465-44"
   2) 1) "rider"
      2) "Roglic"
      3) "speed"
      4) "24.6"
      5) "position"
      6) "45"
      7) "location_id"
      8) "3"
2) 1) "1754326272465-43"
   2) 1) "rider"
      2) "Vingegaard"
      3) "speed"
      4) "34.4"
      5) "position"
      6) "44"
      7) "location_id"
      8) "3"
3) 1) "1754326272465-42"
   2) 1) "rider"
      2) "Vingegaard"
      3) "speed"
      4) "30.8"
      5) "position"
      6) "43"
      7) "location_id"
      8) "1"
4) 1) "1754326272465-41"
   2) 1) "rider"
      2) "Van Aert"
      3) "speed"
      4) "29.2"
      5) "position"
      6) "42"
      7) "location_id"
      8) "1"
5) 1) "1754326272465-40"
   2) 1) "rider"
      2) "Van Aert"
      3) "speed"
      4) "32.6"
      5) "position"
      6) "41"
      7) "location_id"
      8) "1"
```

## Edge Case: When Sequence is 0
:::warning Important Note

If the last stream ID ends in -0 (e.g., 1754326272465-0), you cannot simply subtract 1 from the sequence.

Instead, you must subtract 1 from the timestamp portion and set a high sequence number (e.g., 9999) to access the previous entry block.

Example:
```
Current ID: 1754326272465-0
Next XREVRANGE start ID: 1754326272464-9999
```
Command
```
XREVRANGE race:france 1754326272464-9999 - COUNT 5
```
:::

By chaining XREVRANGE with decremented IDs (handling -0 carefully), you can paginate backward through Redis Streams in a controlled manner.


## Spring Data Redis
Create spring boot project with following commands --OR-- optionally reuse base the previous project by append the controller class
```
spring init -d=web,data-redis,devtools,thymeleaf,lombok \
  -g com.example \
  -a demo \
  -p jar \
  --build maven \
  xrevrange -x
```

add the following to application.yaml
```
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

from the DemoApplication.java
```java
import lombok.RequiredArgsConstructor;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.domain.Range;
import org.springframework.data.redis.connection.stream.MapRecord;
import org.springframework.data.redis.core.StreamOperations;
import org.springframework.data.redis.connection.Limit;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

}

@RestController
@RequestMapping("/api/race/desc")
@RequiredArgsConstructor
class XRevRangeRaceStreamController {

    private final StringRedisTemplate redisTemplate;

    @GetMapping("/paginate")
    public List<MapRecord<String, Object, Object>> paginateStream(
            @RequestParam(defaultValue = "race:france") String streamKey,
            @RequestParam(required = false) String lastSeenId,
            @RequestParam(defaultValue = "5") int count
    ) {
        StreamOperations<String, Object, Object> streamOperations = redisTemplate.opsForStream();
        if (lastSeenId == null) {
            // Initial request - get most recent entries
            return streamOperations.reverseRange(
                streamKey,
                Range.unbounded(),
                Limit.limit().count(count)
            );
        } else {
            return streamOperations.reverseRange(
                streamKey,
                Range.open("-", lastSeenId),  // Define the range from 'endId' down to 'startId'
                Limit.limit().count(count)
            );
        }

    }
}
```


## Example: curl to fetch latest 5 entries from 50 - 46
```
curl "http://localhost:8080/api/race/desc/paginate?streamKey=race:france&count=5" | jq
```

in result
```
[
  {
    "stream": "race:france",
    "value": {
      "rider": "Roglic",
      "speed": "20.3",
      "position": "50",
      "location_id": "3"
    },
    "id": {
      "value": "1754326272465-49",
      "timestamp": 1754326272465,
      "sequence": 49
    },
    "requiredStream": "race:france"
  },
  {
    "stream": "race:france",
    "value": {
      "rider": "Fernandez",
      "speed": "22.5",
      "position": "49",
      "location_id": "5"
    },
    "id": {
      "value": "1754326272465-48",
      "timestamp": 1754326272465,
      "sequence": 48
    },
    "requiredStream": "race:france"
  },
  {
    "stream": "race:france",
    "value": {
      "rider": "Lopez",
      "speed": "20.8",
      "position": "48",
      "location_id": "4"
    },
    "id": {
      "value": "1754326272465-47",
      "timestamp": 1754326272465,
      "sequence": 47
    },
    "requiredStream": "race:france"
  },
  {
    "stream": "race:france",
    "value": {
      "rider": "Moreira",
      "speed": "37.3",
      "position": "47",
      "location_id": "4"
    },
    "id": {
      "value": "1754326272465-46",
      "timestamp": 1754326272465,
      "sequence": 46
    },
    "requiredStream": "race:france"
  },
  {
    "stream": "race:france",
    "value": {
      "rider": "Van Aert",
      "speed": "27.1",
      "position": "46",
      "location_id": "5"
    },
    "id": {
      "value": "1754326272465-45",   <------ last id from previous response
      "timestamp": 1754326272465,
      "sequence": 45
    },
    "requiredStream": "race:france"
  }
]
```

Subsequent request (using the last ID from previous response)
```
curl "http://localhost:8080/api/race/desc/paginate?streamKey=race:france&lastSeenId=1754326272465-45&count=5" | jq
```

in result 
```
[
  {
    "stream": "race:france",
    "value": {
      "rider": "Roglic",
      "speed": "24.6",
      "position": "45",
      "location_id": "3"
    },
    "id": {
      "value": "1754326272465-44",
      "timestamp": 1754326272465,
      "sequence": 44
    },
    "requiredStream": "race:france"
  },
  {
    "stream": "race:france",
    "value": {
      "rider": "Vingegaard",
      "speed": "34.4",
      "position": "44",
      "location_id": "3"
    },
    "id": {
      "value": "1754326272465-43",
      "timestamp": 1754326272465,
      "sequence": 43
    },
    "requiredStream": "race:france"
  },
  {
    "stream": "race:france",
    "value": {
      "rider": "Vingegaard",
      "speed": "30.8",
      "position": "43",
      "location_id": "1"
    },
    "id": {
      "value": "1754326272465-42",
      "timestamp": 1754326272465,
      "sequence": 42
    },
    "requiredStream": "race:france"
  },
  {
    "stream": "race:france",
    "value": {
      "rider": "Van Aert",
      "speed": "29.2",
      "position": "42",
      "location_id": "1"
    },
    "id": {
      "value": "1754326272465-41",
      "timestamp": 1754326272465,
      "sequence": 41
    },
    "requiredStream": "race:france"
  },
  {
    "stream": "race:france",
    "value": {
      "rider": "Van Aert",
      "speed": "32.6",
      "position": "41",
      "location_id": "1"
    },
    "id": {
      "value": "1754326272465-40",
      "timestamp": 1754326272465,
      "sequence": 40
    },
    "requiredStream": "race:france"
  }
]

```
