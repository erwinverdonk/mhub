# MHub message server and client

## Introduction

This project provides:
* a simple message broker (`mserver`) for loosely coupling
software components
* accompanying command-line tools (`mclient`) for interacting
with the server and
* a library for communicating with the server using
Javascript.

It can be used as a lightweight Javascript-only alternative to e.g. RabbitMQ or
MQTT.

Specifically, it was built to support the use-cases of FIRST Lego League
tournaments. It's supported by the [Display System](https://github.com/FirstLegoLeague/displaySystem),
and more components are coming.

## Concepts

### Messages

The purpose of an MHub server (`mserver`) is to pass messages between connected
clients.

A *message* consists of a *topic* and optionally *data* and/or *headers*.

The topic of a message typically represents e.g. a command or event, like
`clock:arm` or `twitter:add`. Data can be the countdown time,
tweet to add, etc.
Headers are for more advanced uses of the system, e.g. to support sending
messages between multiple brokers.

### Publish / subscribe (pubsub)

MHub (like many other message busses) supports the concept of publishing
messages and subscribing to topics (aka 'pubsub').

For example, a client that subscribes to all messages with the topic pattern
`twitter:*`, will receive messages published by other clients with topics such
as `twitter:add` or `twitter:remove`.

Any client can both publish messages and subscribe to other messages.
(Note: messages sent by a client will also be received by that client if it
matches one of its subscriptions).

The topic format is technically free-form, and pattern matching is done by
[minimatch](https://www.npmjs.com/package/minimatch). However, it is advised to
follow e.g. [mq-protocols](https://github.com/FirstLegoLeague/mq-protocols) to
ensure compatibility with other systems.

### Nodes and bindings

An MHub server (`mserver`) instance contains one or more *nodes*.

In many cases, the `default` node will suffice for most users, but to allow for
more flexibility on larger systems, it is possible to define additional nodes.

For example, this mechanism can be used to:
* route tweets to an 'unmoderated' node first, pass through some moderator
  system, then send them through to a 'moderated' node (which then can have
  bindings to other nodes that are interested in these tweets, e.g. display
  nodes).
* route a subset of all messages from an mserver on the local network to an
  mserver on the Internet, e.g. for consumption by a public website.
* route team scores to both a video overlay display and dedicated score displays
  in other areas, but only route the show/hide commands of the scores view to
  the video overlay, such that the dedicated displays keep displaying their
  scores.
* create a firehose node that emits all 'non-confidential' messages for display
  on a public screen somewhere (we're targetting tech-events, right?).

For such larger systems, it is a good idea to assign every application instance
in the system (video display controller, video display, pit area display
controller, pit area display, scores subsystem, etc.) its own node.

*Bindings* then make it possible to selectively route certain messages (based on
their topic) between different nodes. A binding (optionally based on a topic
*pattern*, like subscriptions), forwards all (matching) messages from
its source node to its destination node.

These bindings can either directly be specified in the mserver configuration
(for routing between nodes on the same server), or through an external program
such as [mhub-relay](https://github.com/poelstra/mhub-relay) (for routing
between nodes on the same, and/or different servers).

## Basic installation and usage

To install and run the server:
```sh
npm install -g mhub
mserver
```

You'll now have an MHub server listening on port 13900, containing `default` and
`test` nodes.

To verify that it's working, start an `mclient` in listen mode and connect to
e.g. the `test` node to see a 'blib' message every five seconds:
```sh
mclient -n test -l
```

You should now see e.g.:
```
{ topic: 'blib', data: 3, headers: {} }
{ topic: 'blib', data: 4, headers: {} }
etc...
```

Leave the listening mclient running, then start another one to send a simple
test message:
```sh
mclient -t my:topic
```

You'll see it on the listening `mclient` as:
```
{ topic: 'test:topic', data: undefined, headers: {} }
```

Note that the message was sent to the `default` node (because we didn't specify
another one with `-n`). However, because the server is by default configured to
forward all messages with topic pattern `test:*` from that `default` node to the
`test` node, it does appear at the listening mclient started earlier.
You can see this binding in `server.conf.json`. See below for changing it
yourself.

## `mclient` commandline interface

The `mclient` commandline tool can be used to both listen for messages, or to
publish messages. See `mclient -h` for available commandline parameters:

```
$ mclient -h
Listen mode: node ./bin/mclient [-s <host>[:<port>]] [-n <nodename>] -l [-p <topic_pattern>] [-o <output_format>]
Post mode: node ./bin/mclient [-s <host>[:<port>]] [-n <nodename>] -t <topic> [-d <json_data>] [-h <json_headers>]
Pipe mode: node ./bin/mclient [-s <host>[:<port>]] [-n <nodename>] -t <topic> -i <input_format> [-h <json_headers>]

Options:
  --help         Show help
  -s, --socket   WebSocket to connect to                                                             [required]  [default: "localhost:13900"]
  -n, --node     Node to subscribe/publish to, e.g. 'test'                                           [required]  [default: "default"]
  -l, --listen   Select listen mode
  -p, --pattern  Topic subscription pattern as glob, e.g. 'twitter:*'
  -o, --output   Output format, can be: human, text, jsondata, json                                  [default: "human"]
  -t, --topic    Message topic
  -d, --data     Optional message data as JSON object, e.g. '"a string"' or '{ "foo": "bar" }'
  -h, --headers  Optional message headers as JSON object, e.g. '{ "my-header": "foo" }'
  -i, --input    Read lines from stdin, post each line to server. <input_format> can be: text, json
```

### Listening for messages

Using `mclient`, listen mode is initiated with the `-l` parameter.

In the default (human-friendly) format, short messages are printed on a single
line, but larger messages will wrap across multiple lines. See below for options
to change this.

To simply listen for all messages on the `default` topic, use:
```sh
mclient -l
```

To listen for all messages on another node use e.g.:
```sh
mclient -l -n somenode
```

To only receive messages with a certain topic use e.g.:
```sh
mclient -l -n test -p 'ping:*'
```

(Tip: use the bundled `mping` program to do a quick round-trip time measurement.)

By default, all messages are printed in a somewhat human-readable format, which
is not suitable for consumption by other programs. However, a number of output
options are available to simplify this:
* `human` (default): Outputs raw messages, but mostly without quotes
* `text`: Outputs just the data field of a message as text. Useful when
  listening to a single topic that only contains string data. Note: if data is
  an object, it will still be printed in human-readable format, and may thus
  span multiple lines.
* `json`: Outputs the full message as JSON. Every message is guaranteed to be
  printed on its own line, and contains all info in the message (topic, headers
  and data).
* `jsondata`: Outputs just the data field of a message. Every message is
  guaranteed to be printed on its own line. Useful when listening to just a
  single topic, but complex data is used.

### Publishing single messages

Publishing a single message can be done by specifying the `-t` option (and not
passing `-l` nor `-i`).

To publish a message without data to the `default` topic, use e.g.:

```sh
mclient -t test:something
```

Again, the `-n` option can be used to specify a custom node.
```sh
mclient -n test -t ping:request
```

To pass data (and/or headers) to a message, it needs to be specified as JSON.
This means that e.g. a number can be specified directly as e.g. `42`, but a
string needs to be enclosed in double-quotes (e.g. `"something"`).
Note that shells typically parse these quotes too, so they will need to be
escaped.

For example:
```sh
# On *nix shell:
mclient -t my:topic -d '"some string"'
mclient -t my:topic -d '{ "key": "value" }'
# On Windows command prompt:
mclient -t my:topic -d """some string"""
mclient -t my:topic -d "{ ""key"": ""value"" }"
```

### Publishing multiple messages

It is possible to 'stream' messages to the bus by using the `-i` option.
Available input formats are:
* text: Every line of the input is sent as a string in the message's data field.
* json: Every line of the input is parsed as a JSON object, and used as the
  message's data field.

Example to stream tweets into an mserver, using the `tweet` command from
`node-tweet-cli`.
```sh
tweet login
tweet stream some_topic --json | mclient -t twitter:add -i json
```

## Customizing server nodes and bindings

To customize the available nodes and bindings, create a copy of
`server.conf.json`, edit it to your needs and start the server as:
```sh
mserver -c <config_filename>
```

Note: the pattern in a binding can be omitted, in which case everything will
be forwarded.

Note: don't edit the `server.conf.json` file directly, because any changes to it
will be lost when you upgrade `mhub`. Edit a copy of the file instead.

## Using MHub from Javascript

Simply `npm install --save mhub` in your package, then require the client
interface as e.g.:
```js
// ES6
import MClient from "mhub";
// ES5 (commonjs)
var MClient = require("mhub").MClient;
```

Example usage of subscribing to a node and sending a message:
```js
var MClient = require("mhub").MClient;
var client = new MClient("ws://localhost:13900");
client.on("message", function(message) {
	console.log(message.topic, message.data, message.headers);
});
client.on("open", function() {
	client.subscribe("blib"); // or e.g. client.subscribe("blib", "my:*");
	client.publish("blib", "my:topic", 42, { some: "header" });
});
```

When an error occurs (e.g. the connection is lost), the error will be emitted on
the `error` event.

It is possible (and advisable) to also listen for the `close` event to reconnect
(after some time) to the server in case the connection is lost. Note: any
subscriptions will have to be recreated upon reconnection.

For use in the browser, [browserify](http://browserify.org/) is recommended.

API doc for MClient:
```ts
/**
 * FLL Message Server client.
 *
 * Allows subscribing and publishing to MServer nodes.
 *
 * @event open() Emitted when connection was established.
 * @event close() Emitted when connection was closed.
 * @event error(e: Error) Emitted when there was a connection, server or protocol error.
 * @event message(m: Message) Emitted when message was received (due to subscription).
 */
declare class MClient extends events.EventEmitter {
    /**
     * Create new connection to MServer.
     * @param url Websocket URL of MServer, e.g. ws://localhost:13900
     */
    constructor(url: string);

    /**
     * Connect to the MServer.
     * If connection is already active or pending, this is a no-op.
     * Note: a connection is already initiated when the constructor is called.
     */
    connect(): void;

    /**
     * Disconnect from MServer.
     * If already disconnected, this becomes a no-op.
     *
     * Note: any existing subscriptions will be lost.
     */
    close(): void;

    /**
     * Subscribe to a node. Emits the "message" event when a message is received for this
     * subscription.
     *
     * @param nodeName Name of node in MServer to subscribe to (e.g. "default")
     * @param pattern  Optional pattern glob (e.g. "namespace:*"), matches all messages if not given
     * @param id       Optional subscription ID sent back with all matching messages
     */
    subscribe(nodeName: string, pattern?: string, id?: string): Promise<void>;

    /**
     * Publish message to a node.
     *
     * @param nodeName Name of node in MServer to publish to (e.g. "default")
     * @param topic Message topic
     * @param data  Message data
     * @param headers Message headers
     */
    publish(nodeName: string, topic: string, data?: any, headers?: {
        [name: string]: string;
    }): Promise<void>;
    /**
     * Publish message to a node.
     *
     * @param nodeName Name of node in MServer to publish to (e.g. "default")
     * @param message Message object
     */
    publish(nodeName: string, message: Message): Promise<void>;
}
```

API doc for Message (implemented as a class for convenient construction):
```ts
/**
 * Message to be sent or received over MServer network.
 */
declare class Message {
    /**
     * Topic of message.
     * Can be used to determine routing between pubsub Nodes.
     */
    topic: string;

    /**
     * Optional message data, can be null.
     * Must be JSON serializable.
     */
    data: any;

    /**
     * Optional message headers.
     */
    headers: {
        [name: string]: string;
    };

    /**
     * Construct message object.
     *
     * Warning: do NOT change a message once it's been passed to the pubsub framework!
     * I.e. after a call to publish() or send(), make sure to create 'fresh' instances of e.g.
     * a headers object.
     */
    constructor(topic: string, data?: any, headers?: {
        [name: string]: string;
    });
}
```

## Wire protocol

MHub internally uses JSON messages over websockets. The format of these messages
is not formally specified yet, as it is still under development.
If you're interested in the internals though, `src/MClient.ts` probably is the
best place to get started.

## Contributing

All feedback and contributions are welcome!

Be sure to Star the package if you like it, leave me a
mail, submit an issue, etc.

```sh
git clone https://github.com/poelstra/mhub
cd mhub
npm install
npm run watch
# or run `npm run build` for one-time compilation
```

The package is developed in [Typescript](http://www.typescriptlang.org/), which
adds strong typing on top of plain Javascript. It should mostly be familiar to
Javascript developers.

You can edit the `.ts` files with any editor you like, but to profit from things
like code-completion ('IntelliSense'), I recommend using e.g. GitHub's
[Atom Editor](https://atom.io/) using the awesome
[Atom Typescript](https://github.com/TypeStrong/atom-typescript) plugin.
In SublimeText, use e.g.
[Microsoft's Typescript plugin](https://packagecontrol.io/packages/TypeScript).

For other editors, to get automatic compilation and live-reload:

Please run `npm test` (mainly tslint for now, other tests still pending...)
before sending a pull-request.

## TODO

* Split `mclient` into separate `msub` and `mpub` programs, easier to explain.
* Add timeout option to `mping`
* Document `mping` better
* Add reconnect logic to commandline programs and JS lib

## License

Licensed under the MIT License, see LICENSE.txt.

Copyright (c) 2015 Martin Poelstra <martin@beryllium.net>
