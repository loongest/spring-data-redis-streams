---
sidebar_position: 1
---

# Redis Stream Commands

Redis Stream commands are basically cover the following:

## Add Data
| Command                 | Description                                |
| ----------------------- | ------------------------------------------ |
| `XADD`                  | Adds a new entry to a stream.              |

## List or Inspect Data
| Command           | Description                                                       |
| ----------------- | ----------------------------------------------------------------- |
| `XRANGE`          | Lists entries in a given range (by ID), ascending.                |
| `XREVRANGE`       | Lists entries in a given range, descending order.                 |
| `XLEN`            | Returns the number of entries in a stream.                        |

## Inspect Data
| Command           | Description                                                       |
| ----------------- | ----------------------------------------------------------------- |
| `XINFO STREAM`    | Shows information about the stream (length, first/last ID, etc.). |
| `XINFO GROUPS`    | Lists all consumer groups for a stream.                           |
| `XINFO CONSUMERS` | Lists consumers in a given group.                                 |


## Read Data
| Command      | Description                                                           |
| ------------ | --------------------------------------------------------------------- |
| `XREAD`      | Reads one or more entries from a stream (typically for polling).      |


## Delete Data
| Command              | Description                                        |
| -------------------- | -------------------------------------------------- |
| `XDEL`               | Removes one or more entries from a stream.         |
| `XTRIM`              | Trims older stream entries to cap the stream size. |


## Acknowledge / Claim Data
| Command        | Description                                                         |
| -------------- | ------------------------------------------------------------------- |
| `XACK`         | Acknowledges successful processing of an entry by a consumer group. |
| `XAUTOCLAIM`   | Auto-claims unacknowledged messages that exceed an idle time.       |
| `XCLAIM`       | Transfers pending messages from one consumer to another.            |
| `XSETID`       | Changes the last ID in the stream (use with caution).               |

## Non-Standard or Rare Commands
| Command   | Description                                                                  |
| --------- | ---------------------------------------------------------------------------- |
| `XACKDEL` | (Not a real Redis command — likely a custom combination of `XACK` + `XDEL`). |
| `XDELX`   | (Not a real Redis command — possibly a typo or alias for `XDEL`).            |

## Group Operation
| Command      | Description                                                           |
| ------------ | --------------------------------------------------------------------- |
| `XREADGROUP` | Reads entries on behalf of a consumer group (distributed processing). |
| `XGROUP CREATE`         | Creates a new consumer group for a stream. |
| `XGROUP CREATECONSUMER` | Registers a new consumer to a group.       |
| `XGROUP DELCONSUMER` | Deletes a consumer from a consumer group.          |
| `XGROUP DESTROY`     | Deletes a consumer group.                          |
| `XGROUP SETID` | Resets the ID from which the group starts consuming.                |
| `XPENDING`        | Lists pending messages for a group (not yet acknowledged).        |




Reference: [Redis University Stream](https://redis.io/docs/latest/develop/data-types/streams)