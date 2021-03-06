# socket.io-redis

[![Build Status](https://travis-ci.org/socketio/socket.io-redis.svg?branch=master)](https://travis-ci.org/socketio/socket.io-redis)
[![NPM version](https://badge.fury.io/js/socket.io-redis.svg)](http://badge.fury.io/js/socket.io-redis)

## Table of contents

- [How to use](#how-to-use)
- [Compatibility table](#compatibility-table)
- [API](#api)
  - [adapter(uri[, opts])](#adapteruri-opts)
  - [adapter(opts)](#adapteropts)
  - [RedisAdapter](#redisadapter)
    - [RedisAdapter#sockets(rooms: Set<String>)](#redisadaptersocketsrooms-setstring)
    - [RedisAdapter#allRooms()](#redisadapterallrooms)
    - [RedisAdapter#remoteJoin(id:String, room:String)](#redisadapterremotejoinidstring-roomstring)
    - [RedisAdapter#remoteLeave(id:String, room:String)](#redisadapterremoteleaveidstring-roomstring)
    - [RedisAdapter#remoteDisconnect(id:String, close:Boolean)](#redisadapterremotedisconnectidstring-closeboolean)
- [Client error handling](#client-error-handling)
- [Custom client (eg: with authentication)](#custom-client-eg-with-authentication)
- [With ioredis client](#with-ioredishttpsgithubcomluinioredis-client)
  - [Cluster example](#cluster-example)
  - [Sentinel Example](#sentinel-example)
- [Protocol](#protocol)
- [License](#license)

## How to use

```js
const io = require('socket.io')(3000);
const redisAdapter = require('socket.io-redis');
io.adapter(redisAdapter({ host: 'localhost', port: 6379 }));
```

By running socket.io with the `socket.io-redis` adapter you can run
multiple socket.io instances in different processes or servers that can
all broadcast and emit events to and from each other.

So any of the following commands:

```js
io.emit('hello', 'to all clients');
io.to('room42').emit('hello', "to all clients in 'room42' room");

io.on('connection', (socket) => {
  socket.broadcast.emit('hello', 'to all clients except sender');
  socket.to('room42').emit('hello', "to all clients in 'room42' room except sender");
});
```

will properly be broadcast to the clients through the Redis Pub/Sub mechanism.

If you need to emit events to socket.io instances from a non-socket.io
process, you should use [socket.io-emitter](https://github.com/socketio/socket.io-emitter).

## Compatibility table

| Redis Adapter version | Socket.IO server version |
|-----------------------| ------------------------ |
| 4.x                   | 1.x                      |
| 5.x                   | 2.x                      |
| 6.x                   | 3.x                      |

## API

### adapter(uri[, opts])

`uri` is a string like `localhost:6379` where your redis server
is located. For a list of options see below.

### adapter(opts)

The following options are allowed:

- `key`: the name of the key to pub/sub events on as prefix (`socket.io`)
- `host`: host to connect to redis on (`localhost`)
- `port`: port to connect to redis on (`6379`)
- `pubClient`: optional, the redis client to publish events on
- `subClient`: optional, the redis client to subscribe to events on
- `requestsTimeout`: optional, after this timeout the adapter will stop waiting from responses to request (`5000ms`)

If you decide to supply `pubClient` and `subClient`, make sure you use
[node_redis](https://github.com/mranney/node_redis) as a client or one
with an equivalent API.

### RedisAdapter

The redis adapter instances expose the following properties
that a regular `Adapter` does not

- `uid`
- `prefix`
- `pubClient`
- `subClient`
- `requestsTimeout`

### RedisAdapter#sockets(rooms: Set<String>)

Returns the list of socket IDs connected to `rooms` across all nodes. See [Namespace#allSockets()](https://socket.io/docs/v3/server-api/#namespace-allSockets)

```js
const sockets = await io.of('/').adapter.sockets();
console.log(sockets); // a Set containing all the connected socket ids

const sockets = await io.of('/').adapter.sockets(new Set(['room1', 'room2']));
console.log(sockets); // a Set containing the socket ids in 'room1' or in 'room2'

// this method is also exposed by the Server instance
const sockets = io.in('room3').allSockets();
console.log(sockets); // a Set containing the socket ids in 'room3'
```

### RedisAdapter#allRooms()

Returns the list of all rooms.

```js
const rooms = await io.of('/').adapter.allRooms();
console.log(rooms); // a Set containing all rooms (across every node)
```

### RedisAdapter#remoteJoin(id:String, room:String)

Makes the socket with the given id join the room.

```js
try {
  await io.of('/').adapter.remoteJoin('<my-id>', 'room1');
} catch (e) {
  // the socket was not found
}
```

### RedisAdapter#remoteLeave(id:String, room:String)

Makes the socket with the given id leave the room.

```js
try {
  await io.of('/').adapter.remoteLeave('<my-id>', 'room1');
} catch (e) {
  // the socket was not found
}
```

### RedisAdapter#remoteDisconnect(id:String, close:Boolean)

Makes the socket with the given id to get disconnected. If `close` is set to true, it also closes the underlying socket.

```js
try {
  await io.of('/').adapter.remoteDisconnect('<my-id>', true);
} catch (e) {
  // the socket was not found
}
```

## Client error handling

Access the `pubClient` and `subClient` properties of the
Redis Adapter instance to subscribe to its `error` event:

```js
const adapter = require('socket.io-redis')('localhost:6379');
adapter.pubClient.on('error', function(){});
adapter.subClient.on('error', function(){});
```

The errors emitted from `pubClient` and `subClient` will
also be forwarded to the adapter instance:

```js
const io = require('socket.io')(3000);
const redisAdapter = require('socket.io-redis');
io.adapter(redisAdapter({ host: 'localhost', port: 6379 }));
io.of('/').adapter.on('error', function(){});
```

## Custom client (eg: with authentication)

If you need to create a redisAdapter to a redis instance
that has a password, use pub/sub options instead of passing
a connection string.

```js
const redis = require('redis');
const redisAdapter = require('socket.io-redis');
const pub = redis.createClient(port, host, { auth_pass: "pwd" });
const sub = redis.createClient(port, host, { auth_pass: "pwd" });
io.adapter(redisAdapter({ pubClient: pub, subClient: sub }));
```

## With ioredis client

### Cluster example

```js
const io = require('socket.io')(3000);
const redisAdapter = require('socket.io-redis');
const Redis = require('ioredis');

const startupNodes = [
  {
    port: 6380,
    host: '127.0.0.1'
  },
  {
    port: 6381,
    host: '127.0.0.1'
  }
];

io.adapter(redisAdapter({
  pubClient: new Redis.Cluster(startupNodes),
  subClient: new Redis.Cluster(startupNodes)
}));
```

### Sentinel Example

```js
const io = require('socket.io')(3000);
const redisAdapter = require('socket.io-redis');
const Redis = require('ioredis');

const options = {
  sentinels: [
    { host: 'somehost1', port: 26379 },
    { host: 'somehost2', port: 26379 }
  ],
  name: 'master01'
};

io.adapter(redisAdapter({
  pubClient: new Redis(options),
  subClient: new Redis(options)
}));
```

## Protocol

The `socket.io-redis` adapter broadcasts and receives messages on particularly named Redis channels. For global broadcasts the channel name is:
```
prefix + '#' + namespace + '#'
```

In broadcasting to a single room the channel name is:
```
prefix + '#' + namespace + '#' + room + '#'
```


- `prefix`: The base channel name. Default value is `socket.io`. Changed by setting `opts.key` in `adapter(opts)` constructor
- `namespace`: See https://github.com/socketio/socket.io#namespace.
- `room` : Used if targeting a specific room.

A number of other libraries adopt this protocol including:

- [socket.io-emitter](https://github.com/socketio/socket.io-emitter)
- [socket.io-python-emitter](https://github.com/GameXG/socket.io-python-emitter)
- [socket.io-emitter-go](https://github.com/stackcats/socket.io-emitter-go)

## License

MIT
