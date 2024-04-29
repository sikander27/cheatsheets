# Redis Cheat Sheet

Redis is an open-source, in-memory data structure store, used as a database, cache, and message broker. Here are some basic commands to help you get started with the Redis command line interface (CLI).

## Connecting to Redis

To connect to Redis using the CLI:

```
redis-cli
```

## Key-Value Operations

Set a key-value pair:

```
SET key value
```

Get the value of a key:

```
GET key
```

List all keys:

```
KEYS *
```

Delete a key:

```
DEL key
```

Check if a key exists:

```
EXISTS key
```

## Set Operations

List all members of a set:

```
SMEMBERS key
```

## Publish-Subscribe

Publish a message to a channel:

```
PUBLISH channel message
```

Subscribe to a channel:

```
SUBSCRIBE channel
```

Unsubscribe from a channel:

```
UNSUBSCRIBE channel
```

These are some of the basic commands you can use in the Redis CLI. Replace `key`, `value`, and `channel` with your actual keys, values, and channel names.

For more commands and details, refer to the [Redis documentation](https://redis.io/commands).
```

Save this content to a file with a `.md` extension, and you'll have your Redis cheat sheet ready to use!
