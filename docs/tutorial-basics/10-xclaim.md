---
sidebar_position: 10
---

# XCLAIM

We've cover quite a lot so far, let’s now explore how to identify and handle failed entries (i.e., pending messages that were never acknowledged due to errors or consumer crashes).

These messages stay in the Pending Entries List (PEL) until they are:
* Successfully acknowledged (XACK)
* Reclaimed via XCLAIM or XAUTOCLAIM
* Expire due to a TTL policy (Redis doesn’t do this automatically)

## View Failed / Stuck Pending Entries with XPENDING
```
XPENDING race:france race-group - + 10
```
This lists the first 10 pending messages, with details:
```
1) 1) "1754490516011-21"     # Stream ID
   2) "consumer-B"           # Assigned consumer
   3) (integer) 45000        # Idle time in ms (e.g., 45 seconds)
   4) (integer) 1            # Delivery count
```
__Use this to:__
* Identify which entries are stuck
* See how long they’ve been pending (idle time)
* See how many times they’ve been delivered (delivery count)

## Automatically Claim Stuck Messages with XAUTOCLAIM
Syntax
```
XAUTOCLAIM <stream> <group> <consumer> <min-idle-time> <start-id> [COUNT n]
```
Example
```
XAUTOCLAIM race:france race-group consumer-B 30000 0-0 COUNT 10
```
This means:
    * Reassign messages idle for more than 30 seconds
    * Starting from the oldest message (0-0)
    * Reassign them to consumer-B
    * Max 10 entries per call

➡ You’ll receive a list of messages to reprocess + the new start ID.

## Spring Data Redis Example: Auto-Reclaim Failed Messages

Create spring boot project with following commands, this time we'll use h2 and Spring data JPA as well to process the data.
```
spring init -d=web,data-redis,devtools,thymeleaf,lombok \
  -g com.example \
  -a demo \
  -p jar \
  --build maven \
  xclaim -x
```

add the following to application.yaml
```
spring:
  data:
    redis:
      host: localhost
      port: 6379
  h2:
    console:
      enabled: true
      path: /h2-console
  datasource:
    name: example
    username: sa
    password:
    generate-unique-name: false
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  sql:
    init:
      mode: never
  jpa:
    open-in-view: false
    show-sql: true
    generate-ddl: true
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
        format_sql: true
        generate_statistics: true
    hibernate:
      ddl-auto: create-drop

logging:
  level:
    root: INFO
    org:
      springframework:
        security: TRACE
        web:
          reactive:
            function:
              client: DEBUG
      hibernate:
        SQL: DEBUG
        orm:
          jdbc:
            bind: TRACE
stream:
  idle-timeout-seconds: 900 # 15 minutes

```
There is no standard fixed value for how long to set idle-timeout-seconds — it entirely depends on how you consume and process your data.

You should set the timeout based on the maximum expected processing time for a message in your system, with some extra buffer to avoid premature recovery.


### Example
In my case, if the service is responsible for parsing large Excel files, and each job typically takes around 5 to 10 minutes, then I would configure a slightly longer timeout — say, 15 minutes — to safely consider the message as "stuck" if it hasn’t been acknowledged in that time.

application.yaml
```
stream:
  idle-timeout-seconds: 900 # 15 minutes
```
This allows the system to tolerate long processing times without falsely assuming a failure — while still enabling automatic recovery for truly stalled consumers.

### Entity and Repository
Entity - RaceStreamEntity.java
```java
@Entity
@Table(name = "race_stream_entry")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class RaceStreamEntity {

    @Id
    @GeneratedValue
    @UuidGenerator
    @Setter(AccessLevel.NONE)
    private UUID id;

    @Column(name = "stream_id", unique = true)
    private String streamId; // Redis Stream ID (e.g., 1754326272465-45)

    @Column(name = "rider")
    private String rider;

    @Column(name = "speed")
    private Double speed;

    @Column(name = "position")
    private Integer position;

    @Column(name = "location_id")
    private Integer locationId;

    @Column(name = "received_at")
    private LocalDateTime receivedAt;

}
```
Repository - RaceStreamRepository.java
```java
@Repository
public interface RaceStreamRepository extends JpaRepository<RaceStreamEntity, UUID>, JpaSpecificationExecutor<RaceStreamEntity> {

    boolean existsByStreamId(String streamId);

}
```


### StreamRecoveryService
Next, we create a new class in Spring Boot using StreamCommands.xAutoClaim() (low-level access):
```java
@Slf4j
@Service
@RequiredArgsConstructor
public class StreamRecoveryService {

    private final StringRedisTemplate redisTemplate;
    private final RaceStreamRepository repository;

    private static final String STREAM_KEY = "race:france";
    private static final String GROUP_NAME = "race-group";
    // A dedicated consumer name for handling recovered messages
    private static final String RECOVERY_CONSUMER_NAME = "recovery-consumer";
    private final StreamOperations<String, Object, Object> streamOps = redisTemplate.opsForStream();

    @Value("${stream.idle-timeout-seconds:900}")
    private long idleTimeoutSeconds;

    /**
     * Ensures the consumer group exists at startup.
     * This is idempotent and safe to run every time.
     */
    @PostConstruct
    private void init() {
        try {
            // Equivalent to: XGROUP CREATE race:france race-group 0 MKSTREAM
            redisTemplate.opsForStream().createGroup(STREAM_KEY, ReadOffset.from("0"), GROUP_NAME);
            log.info("✅ Created or verified group '{}' on stream '{}'", GROUP_NAME, STREAM_KEY);
        } catch (RedisSystemException e) {
            // If the group already exists, Redis throws a BUSYGROUP error. We can safely ignore it.
            if (e.getCause() instanceof RedisBusyException) {
                log.info("ℹ️ Group '{}' already exists, skipping creation.", GROUP_NAME);
            } else {
                log.error("❌ Could not initialize Redis Stream group", e);
                throw e;
            }
        }
    }


    /**
     * This scheduled task runs periodically to find and reclaim messages that have been
     * pending for too long (i.e., were delivered to a consumer that crashed or failed
     * to acknowledge them). This prevents messages from getting stuck in the stream.
     */
    @Scheduled(fixedRate = 60_000) // Run every 60 seconds
    public void reclaimStuckMessages() {
        log.info("⚙️ Running job to reclaim stuck messages...");

        try {
            // 1. Check the pending messages for the entire group.
            // This is equivalent to: XPENDING race:france race-group
            PendingMessagesSummary summary = streamOps.pending(STREAM_KEY, GROUP_NAME);
            if (summary == null || summary.getTotalPendingMessages() == 0) {
                log.info("✅ No pending messages to reclaim.");
                return;
            }

            log.info("ℹ️ Found {} total pending messages. Checking for idle ones...", summary.getTotalPendingMessages());

            // 2. Find messages that have been idle for more than 30 seconds.
            // We check pending messages for any consumer ('-') up to a certain count ('+').
            PendingMessages pendingMessages = streamOps.pending(
                    STREAM_KEY,
                    GROUP_NAME,
                    Range.unbounded(), // Check all pending messages
                    100L // Limit to checking 100 at a time to avoid overload
            );

            if (pendingMessages.isEmpty()) {
                return;
            }

            // 3. Iterate and claim messages that are idle for too long.
            for (PendingMessage pendingMessage : pendingMessages) {
                if (pendingMessage.getElapsedTimeSinceLastDelivery().compareTo(Duration.ofSeconds(idleTimeoutSeconds)) > 0) {
                    log.warn("🚨 Message {} from consumer '{}' has been idle for {}. Reclaiming...",
                            pendingMessage.getId(), pendingMessage.getConsumerName(), pendingMessage.getElapsedTimeSinceLastDelivery());

                    // 4. Claim the message. This changes its ownership to our recovery consumer.
                    // Equivalent to: XCLAIM race:france race-group recovery-consumer 30000 <message-id>
                    List<MapRecord<String, Object, Object>> claimedRecords = streamOps.claim(
                            STREAM_KEY,
                            GROUP_NAME,
                            RECOVERY_CONSUMER_NAME,
                            Duration.ofSeconds(30), // min-idle-time for the claim to succeed
                            pendingMessage.getId()
                    );

                    // 5. Process the now-claimed message.
                    if (!claimedRecords.isEmpty()) {
                        processAndAcknowledge(claimedRecords);
                    }
                }
            }

        } catch (DataAccessException e) {
            log.error("❌ Redis access error during message reclamation: {}", e.getMessage(), e);
        } catch (Exception e) {
            log.error("❌ An unexpected error occurred during message reclamation", e);
        }
    }

    /**
     * Processes a list of records, saves them to the database, and acknowledges them.
     * This logic can be shared by the main consumer and the recovery service.
     * @param records The list of MapRecord to process.
     */
    private void processAndAcknowledge(List<MapRecord<String, Object, Object>> records) {
        if (records == null || records.isEmpty()) {
            return;
        }

        for (MapRecord<String, Object, Object> record : records) {
            String streamId = record.getId().getValue();
            try {
                // Idempotency check: only process if not already saved.
                if (!repository.existsByStreamId(streamId)) {
                    RaceStreamEntity entity = toEntity(record);
                    repository.save(entity);
                    log.info("✅ Successfully re-processed and saved message {}", streamId);

                    // Acknowledge the message to remove it from the Pending Entries List (PEL).
                    streamOps.acknowledge(STREAM_KEY, GROUP_NAME, record.getId());
                    log.info("👍 Acknowledged message {}", streamId);
                } else {
                    // If it's already in the DB, it means a previous attempt succeeded
                    // but failed to acknowledge. We can now safely acknowledge it.
                    log.warn("⚠️ Message {} was already processed. Acknowledging now.", streamId);
                    streamOps.acknowledge(STREAM_KEY, GROUP_NAME, record.getId());
                }
            } catch (Exception e) {
                log.error("❌ Failed to process reclaimed message {}: {}", streamId, e.getMessage(), e);
                // Decide on a retry strategy or move to a dead-letter queue if needed.
            }
        }
    }

    /**
     * Maps a Redis Stream record to a JPA entity.
     */
    private RaceStreamEntity toEntity(MapRecord<String, Object, Object> record) {
        Map<Object, Object> valueMap = record.getValue();
        return RaceStreamEntity.builder()
                .streamId(record.getId().getValue())
                .rider(valueMap.getOrDefault("rider", "").toString())
                .speed(Double.parseDouble(valueMap.getOrDefault("speed", "0").toString()))
                .position(Integer.parseInt(valueMap.getOrDefault("position", "0").toString()))
                .locationId(Integer.parseInt(valueMap.getOrDefault("location_id", "0").toString()))
                .receivedAt(LocalDateTime.now()) // Or parse from a timestamp in the message
                .build();
    }
}
```
You can call this method periodically (e.g., in a scheduled task) to recover stuck messages.

## Key Takeaways
| Task                    | Tool/Command                      |
| ----------------------- | --------------------------------- |
| List all stuck messages | `XPENDING stream group - + COUNT` |
| Reassign idle messages  | `XAUTOCLAIM`                      |
| Retry and reprocess     | Use message ID + logic            |
| Remove after success    | `XACK`                            |


## Step-by-Step: Simulate Stuck Message for Recovery
__1. Start Fresh: Flush DB__
```
127.0.0.1:6379> FLUSHDB
```

__2. Generate 50 Messages with Lua Script__
```
redis-cli --eval generate_race.lua race:france , 50
```

__3. Create Consumer Group__
```
127.0.0.1:6379> XGROUP CREATE race:france race-group 0 MKSTREAM
```
This sets up the group from the beginning of the stream.

__4. Simulate a consumer processing 10 messages__
```
127.0.0.1:6379> XREADGROUP GROUP race-group consumer-B COUNT 10 STREAMS race:france >
```
This will:
* Deliver the first 10 messages to consumer-B
* Pending state starts for these 10 entries
* No XACK yet = Redis considers them unacknowledged

__5. Now simulate consumer crash__
* Just do nothing — leave the messages unacknowledged
* Close your terminal
* Wait 30 minutes (or whatever duration you want to simulate as idle)

__6. Now run your recovery application__

When your Spring Boot application starts, it should periodically check for stuck messages using a recovery scheduler like:
```
if (pending.getElapsedTimeSinceLastDelivery().compareTo(Duration.ofSeconds(idleTimeoutSeconds)) > 0) {
    // This message is considered stuck → claim it
}
```
These messages can now be claimed, reprocessed, and acknowledged.

__7. Check Pending Messages (Optional)__
You can inspect the pending entries via:
```
XPENDING race:france race-group

or 

XPENDING race:france race-group - + 10
```
This gives:
```
1) (integer) 10
2) "1754496428424-0"
3) "1754496428424-9"
4) 1) 1) "consumer-B"
      2) "10"

or 

127.0.0.1:6379> XPENDING race:france race-group - + 10
 1) 1) "1754496428424-0"
    2) "consumer-B"
    3) (integer) 250220     # Idle time in ms (e.g., 25 seconds)
    4) (integer) 1
 2) 1) "1754496428424-1"
    2) "consumer-B"
    3) (integer) 250220
    4) (integer) 1
```

__8. Lower Idle Timeout for Testing__
To speed up simulation, change the idle timeout config from 15 minutes to just 2 minutes:
```
# application.yaml
stream:
  idle-timeout-seconds: 120 # 2 minutes
```

Then restart your Spring Boot app and observe the recovery logs:
```
..StreamRecoveryService   : ✅ Successfully re-processed and saved message 1754496428424-9
..StreamRecoveryService   : 👍 Acknowledged message 1754496428424-9
..StreamRecoveryService   : ⚙️ Running job to reclaim stuck messages...
..StreamRecoveryService   : ✅ No pending messages to reclaim.
```
With this setup, you've successfully simulated and verified message recovery from an idle Redis Stream consumer.