# Part 3: Prompt Preparation

## Selected PR: [#193 — offsets_for_times, beginning_offsets, end_offsets on AIOKafkaConsumer](https://github.com/aio-libs/aiokafka/pull/193)

---

## 3.1.1 Repository Context

`aiokafka` is a Python library that provides an asyncio-compatible client for Apache Kafka. Kafka itself is a distributed log system — producers write records to named topics, and consumers read those records in order, tracking their position with numeric offsets. The challenge for Python developers is that the dominant Kafka client ecosystem was originally Java-first, and naive Python wrappers tended to block the event loop during network calls, making them unusable in async web frameworks.

`aiokafka` solves this by wrapping `kafka-python`'s protocol parsing layer with coroutine-based I/O so that Kafka operations — connecting to brokers, fetching records, committing offsets, rejoining consumer groups after rebalances — can all happen without blocking the asyncio event loop. The two central classes are `AIOKafkaConsumer`, which reads messages from topics, and `AIOKafkaProducer`, which publishes them.

The intended users are Python backend developers building streaming data pipelines, event-driven microservices, or real-time processing systems — places where you might be using FastAPI or aiohttp as your web layer and want to consume Kafka messages in the same process without spawning threads. The library also handles the complexity of consumer group coordination: when multiple instances of a service share the same `group_id`, Kafka assigns each instance a disjoint set of partitions, and `aiokafka` manages the heartbeat, rebalance, and offset-commit lifecycle automatically in background tasks.

The problem domain is distributed, at-least-once (or exactly-once, with transactions) message processing. Reliability and correctness matter a great deal here — a bug in offset management can cause messages to be replayed or skipped silently, which is a serious problem in financial, logging, or audit use cases.

---

## 3.1.2 Pull Request Description

This PR adds three methods to `AIOKafkaConsumer` that allow a caller to query Kafka for partition offsets without changing the consumer's read position:

- `offsets_for_times(timestamps)` — takes a mapping of `TopicPartition` to UNIX timestamps in milliseconds and returns, from the broker, the earliest message offset whose timestamp is at or after each given value.
- `beginning_offsets(partitions)` — returns the log-start offset (the oldest retained message offset) for the given partitions.
- `end_offsets(partitions)` — returns the log-end offset (one past the last written message) for the given partitions.

**Why were these changes needed?**

Before this PR, there was no clean way to replay messages from a point in time using `aiokafka`. The Java Kafka client had offered `offsetsForTimes()` since Kafka 0.10, and it was a commonly requested feature (tracked in issue #164). Users who needed to reprocess events from a specific timestamp had to either drop into the lower-level `kafka-python` sync client — breaking the async abstraction — or binary-search through offsets manually, which was slow and error-prone.

**Previous behaviour vs. new behaviour:**

Previously, the only ways to control where consumption started were `seek()` (to an absolute offset), `seek_to_beginning()` (log start), and `seek_to_end()` (log end). After this PR, a consumer can query: "What is the first offset in partition 0 at or after 2017-09-01T00:00:00 UTC?" and then `seek()` to that offset, making time-based replay a first-class operation. The `beginning_offsets` and `end_offsets` additions also let code inspect partition bounds without moving the read cursor, which is useful for lag monitoring and progress calculations.

---

## 3.1.3 Acceptance Criteria

✓ When `consumer.offsets_for_times({tp: timestamp_ms})` is called on a broker >= 0.10 with a valid timestamp, the returned dict maps `tp` to an `OffsetAndTimestamp(offset, timestamp)` namedtuple where the offset's message timestamp is the earliest one >= the supplied timestamp.

✓ When `offsets_for_times` is called on a broker < 0.10 (which does not support `ListOffsetsRequest` v1), the method must raise `UnsupportedVersionError` rather than returning incorrect data or hanging indefinitely.

✓ When `consumer.beginning_offsets(partitions)` is called, it must return the current log-start offset for each partition without modifying the consumer's internal fetch position — confirmed by asserting `consumer.position(tp)` is unchanged before and after the call.

✓ When `consumer.end_offsets(partitions)` is called, it must return a value that is at least as large as the number of messages produced to that partition so far, and the method must not trigger a consumer group rebalance or emit a heartbeat failure.

✓ The implementation should handle the case where a partition has no messages at or after the requested timestamp by returning `None` for that partition (not raising an exception), matching the Java client's `offsetsForTimes` contract.

✓ All three methods must work on partitions that the consumer has not been explicitly assigned to — they are read-only metadata queries and should not require a prior `assign()` or `subscribe()` call for the target partitions.

✓ Integration tests must pass against a real Kafka broker (dockerized in the CI environment) for all three methods, covering happy paths, old-broker rejection, and unassigned-partition calls.

---

## 3.1.4 Edge Cases

**Edge Case 1: Timestamp before all retained messages**

If the timestamp passed to `offsets_for_times` is older than the earliest message the broker has retained (e.g., due to retention policy cleaning up old segments), the method should return the beginning offset for that partition, not `None`. This can confuse callers who assume `None` means "no data" — in reality it means "no data at or after this timestamp," but data does exist. The implementation should at minimum be consistent with the Java client's behaviour here, and the documentation should clarify this edge case explicitly.

**Edge Case 2: Concurrent consumer group rebalance**

If a rebalance is in progress when `offsets_for_times` or `end_offsets` is called, the broker assignment may be shifting. The methods should either wait for the rebalance to complete or explicitly document that they can be called for any partition (not just currently assigned ones), so callers can design around this safely. A test should verify that calling `offsets_for_times` on a partition that is in the middle of being reassigned does not corrupt the consumer's group state.

**Edge Case 3: Network timeout during the ListOffsets request**

If the broker does not respond to `ListOffsetsRequest` within `request_timeout_ms`, the method should raise `KafkaTimeoutError` and not leave any half-completed state in the client. The caller should be able to retry safely after this error. This matters particularly for `end_offsets` used in consumer lag monitors, where timeouts must be surfaced rather than silently swallowed.

**Edge Case 4: Empty partition list**

Calling any of the three methods with an empty collection should return an empty dict immediately, not send a network request. This prevents unnecessary broker calls in code that conditionally builds partition lists.

---

## 3.1.5 Initial Prompt

```
You are implementing three new methods on the `AIOKafkaConsumer` class in the `aiokafka` Python library.

## Background

`aiokafka` (https://github.com/aio-libs/aiokafka) is an asyncio-based Kafka client for Python. It wraps `kafka-python`'s protocol layer with async coroutines so that Kafka I/O does not block the event loop. The central consumer class is `AIOKafkaConsumer` in `aiokafka/consumer/consumer.py`.

Apache Kafka brokers >= 0.10 support a `ListOffsetsRequest` v1 API that allows clients to look up the earliest message offset at or after a given UNIX timestamp. The Java Kafka client exposes this as `offsetsForTimes()`. aiokafka does not currently have an equivalent, which is tracked in issue #164.

## What you need to implement

Add three coroutine methods to `AIOKafkaConsumer`:

### 1. `async def offsets_for_times(self, timestamps)`
- `timestamps`: `dict[TopicPartition, int]` — mapping of partition to UNIX timestamp in milliseconds
- Returns: `dict[TopicPartition, OffsetAndTimestamp | None]`
- Sends `ListOffsetsRequest` v1 to the leader broker for each partition
- If the broker version is < 0.10 (does not support v1), raise `UnsupportedVersionError`
- If no message exists at or after the timestamp for a partition, return `None` for that key
- Must NOT modify the consumer's internal fetch position

### 2. `async def beginning_offsets(self, partitions)`
- `partitions`: iterable of `TopicPartition`
- Returns: `dict[TopicPartition, int]`
- Uses `ListOffsetsRequest` with the sentinel value `EARLIEST_TIMESTAMP = -2`
- If broker version < 0.10, raise `UnsupportedVersionError`
- Must NOT call `seek_to_beginning()` or modify fetch position

### 3. `async def end_offsets(self, partitions)`
- Same structure as `beginning_offsets` but uses sentinel `LATEST_TIMESTAMP = -1`
- Returns the log-end offset (one past the last written message index)

## Acceptance criteria to satisfy (reference these when writing tests)

- `offsets_for_times` returns correct `OffsetAndTimestamp` for a known timestamp on broker >= 0.10
- Old broker (< 0.10) raises `UnsupportedVersionError` for all three methods
- `beginning_offsets` and `end_offsets` do not change `consumer.position(tp)` for any partition
- Calling with an empty dict/list returns an empty dict without making a network request
- Network timeout raises `KafkaTimeoutError`; caller can retry safely
- Methods work on partitions not currently assigned to this consumer

## Edge cases to handle explicitly

1. Timestamp earlier than all retained messages → return beginning offset, not None
2. Empty input collection → return `{}` immediately, no broker request
3. Request timeout → raise `KafkaTimeoutError`, leave no partial state
4. Rebalance in progress → the methods should not depend on current group assignment

## Implementation guidance

- Look at how `kafka-python`'s `KafkaConsumer.offsets_for_times()` constructs `ListOffsetsRequest` (in `kafka/consumer/consumer.py`) — reuse the same request-building logic, but await the response asynchronously
- The `AIOKafkaClient` already has a `send()` method for dispatching arbitrary Kafka requests; use it
- `OffsetAndTimestamp` should be a namedtuple with fields `(offset, timestamp)`, matching the Java client's return type
- Add tests in `tests/test_consumer.py` that run against the dockerized Kafka broker already set up in the test suite

## Files to modify

- `aiokafka/consumer/consumer.py` — add the three methods
- `aiokafka/structs.py` — add `OffsetAndTimestamp` namedtuple if not already present
- `aiokafka/client.py` — add helpers for constructing and sending `ListOffsetsRequest` if needed
- `tests/test_consumer.py` — add integration tests
- `docs/consumer.rst` — document the new APIs with a usage example

Please write well-commented, async-idiomatic Python code. All public methods must have docstrings that match the style in the existing codebase. Raise specific exception types (`UnsupportedVersionError`, `KafkaTimeoutError`) rather than generic ones.
```

---

*I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words.*
